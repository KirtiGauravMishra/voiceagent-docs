# Backend Prompt 03 — Calls & the Real-Time Voice Pipeline

> Read `../00-master-rebuild-guide.md`, `01-auth.md`, and `02-agents.md` first. This is the most complex and most important module in the system — budget the most build time here, and **design it multi-instance-from-day-one** (see §7 below and the master guide §2). Do not build this as an in-memory single-process prototype "to get it working first" — the in-memory shortcuts described as "what the reference implementation did" below are exactly what you must **not** copy; build the Redis-backed version directly.

## Goal
Every phone call (manual, inbound, campaign-dialed, or workflow-triggered) is one `Call` document driven through Twilio. The AI conversation happens over a **real-time WebSocket media stream** (Deepgram STT + streaming LLM + low-latency TTS with true barge-in) — this is the primary path; a turn-based Twilio-`<Gather>` fallback is acceptable only when Deepgram isn't configured.

## Data model — `Call`

`agentId` (→Agent, required), `agentName` (string, denormalized copy of the agent's name at the moment the call is created — keeps this call's detail view sensible even after the agent is later renamed or deleted, see Prompt 02's deletion policy), `userId` (→User, required, owner), `contactId` (→Contact, optional), `campaignId`/`campaignContactId` (set only for campaign-dialed calls), `status` (enum `queued|ringing|in-progress|completed|failed|no-answer|busy|voicemail`), `direction` (`inbound|outbound`), `from`/`to` (E.164), `twilioCallSid`, `recordingUrl`/`recordingSid`, `durationSeconds` (see §6 — resolved from multiple sources, not set by one webhook), `transcript` (flattened string), `transcriptEntries` (`{role:'agent'|'user', text, timestamp}[]`, appended turn-by-turn live), `sentiment` (`positive|neutral|negative`), `callScore` (0–100), `summary`/`extractedData`, `webhookSent` (bool), `transferred` (bool — set true the moment a human-handoff `<Dial>` actually fires, see §"Human handoff" below), `appointmentBooked` (bool, default `false`), `appointmentAt` (Date, optional — the actual booked slot, set only on a confirmed Cal.com booking response, never optimistically), `workflowFollowup` (bool — true if this call **is itself** an automation follow-up, prevents infinite trigger loops), `workflowTriggered` (bool — atomic one-shot flag so the completion trigger can't double-fire from a race between two webhooks), `startedAt`/`completedAt`.

Indexes: `{agentId,createdAt:-1}`, `{userId,createdAt:-1}`, `{agentId,status,createdAt:-1}`, `{campaignId,createdAt:-1}`, `{twilioCallSid:1}` sparse, `{status:1,createdAt:1}` partial on `status:'in-progress'` (powers the stuck-call cleanup sweep, Prompt 11).

## Dispatching a call — `dispatchCall(userId, agentId, to, direction, contactId?, workflowFollowup?)`

1. Load the agent, check ownership and `status !== 'inactive'` (403 otherwise). **No quota/billing gate** — there's no plan system in this rebuild; only check the user isn't `suspended`.
2. **Caller-ID resolution**:
   - **Workflow-automation calls** (`workflowFollowup:true`) must **never** use a shared/platform default number: try agent's `callSettings.phoneNumberId` → owner's default active non-shared `PhoneNumber` → any active non-shared number owned by the user. If none exists, throw `400 WORKFLOW_CALLER_ID` telling the user to add a number. Automated calls must always go out under this org's own number.
   - **Manual calls**: `TWILIO_PHONE_NUMBER` env default → agent's `phoneNumberId` → owner's default active number.
3. Create the `Call` row (`status:'queued'`), bump `Contact.totalCalls`/`lastCalledAt` if linked.
4. Place the real Twilio call via `initiateOutboundCall`.

Validate the phone number with `/^\+?[1-9]\d{6,14}$/` after normalizing out dashes/spaces/parens.

## API

**`/api/calls`** (also mountable at `/api/agents/:agentId/calls`), JWT required:
| Method | Path | Purpose |
|---|---|---|
| POST | `/` | dispatch — rate-limit generously (Redis-backed, see master guide; do not hardcode a low per-IP limit — this endpoint needs to sustain the platform's full call volume) |
| GET | `/` | list caller's calls (or agent-scoped when nested), paginated |
| GET | `/:id` | single call detail |
| POST | `/:id/complete` | manual completion — see §6 |

**`/api/twilio`** — all **public** (no JWT; Twilio calls these directly). **Every handler must always return HTTP 200 with valid TwiML**, even on internal error — build a fallback-TwiML helper (`<Say>...</Say><Hangup/>`) and use it in every catch block; a non-200/malformed response makes Twilio play a jarring generic error tone to a live caller. **Implement Twilio signature validation (`X-Twilio-Signature`) on every one of these routes** — required, not optional.

| Method | Path | Purpose |
|---|---|---|
| GET/POST | `/voice/:callId` | initial TwiML when the call connects |
| GET | `/greeting-audio/:callId` | serves pre-generated greeting audio (regenerate on-demand, don't read from local disk — see §7) |
| POST | `/gather/:callId` | turn-based loop, Gather-fallback mode only |
| POST | `/status/:callId` | call status changes |
| POST | `/recording/:callId` | recording ready |
| POST | `/inbound/:phoneNumberId` | inbound call to an owned number |
| POST | `/inbound` | unrecognized-number fallback |
| GET | `/recording-proxy?url=` | proxy a Twilio recording (Basic-Auth) to the frontend, injecting credentials server-side. **Validate that `url` is actually a Twilio-owned domain before proxying** — don't build an open proxy. |

## Initial TwiML (`buildInitialTwiml`)

1. Load `Call` + `Agent`.
2. Resolve the greeting (`callSettings.greetingMessage` or a default).
3. Map the agent's voice to the nearest native Twilio Polly voice for `<Say>` (table below) — used even for `openai`-provider agents so the initial TwiML round-trip doesn't need a pre-generated file.
4. If the voice provider needs pre-generated audio (`cartesia`/`elevenlabs`) and Deepgram is configured (streaming mode), fetch the greeting via the `/greeting-audio/:callId` **route** (regenerated on demand) rather than reading a locally-written file — required so any instance behind a load balancer can serve it.
5. If `DEEPGRAM_API_KEY` is configured: return `<Connect><Stream url="wss://.../media-stream/:callId"/></Connect>` followed by a silent `<Gather>` fallback + `<Hangup>` (in case the stream never connects). Use a **path-based** stream URL (`/media-stream/<callId>`, not a query string) — some reverse proxies strip query strings on WebSocket upgrades.
6. Otherwise: classic `<Gather input="speech" action=".../gather/:callId" timeout="5" speechTimeout="auto">`.

**Voice mapping table** (OpenAI voice name → Twilio Polly Neural voice for `<Say>`): `alloy→Polly.Matthew-Neural`, `echo→Polly.Stephen-Neural`, `fable→Polly.Brian-Neural`, `onyx→Polly.Gregory-Neural`, `nova→Polly.Joanna-Neural`, `shimmer→Polly.Ruth-Neural`.

## Real-time media-stream pipeline (primary path)

Wire a WebSocket server onto the same HTTP server's `'upgrade'` event, filtering for `/media-stream` paths (don't run a separate server/port).

**Connection setup**: Twilio can send its `start` event within milliseconds — before your async `Call`+`Agent` DB lookup resolves. **Buffer incoming messages** from the instant the socket opens; once the DB lookup (and any Cal.com context resolution, wrapped in try/catch) completes, install the real handler and replay the buffer in order. Don't lose the `start` event.

**Session state — put this in Redis, not a local `Map`** (see §7; this is the single most important architectural deviation from a naive implementation): conversation `history`, loaded agent config, `isSpeaking`/`isSpeakingUntil` (barge-in state), an abort mechanism + incrementing turn/pipeline ID (so a new turn can cleanly supersede a stale in-flight one), and which Twilio-owned number to use as caller ID on a human-transfer `<Dial>` (the number that received the call for inbound, the caller-ID number for outbound — Twilio requires the `<Dial>` `callerId` to be owned by the account placing the bridge).

**If Redis isn't reachable when a session should be created**: don't let the WebSocket handler throw/crash — fall back to the same fallback-TwiML helper used elsewhere (apology + hangup) and mark the `Call` `failed` with a clear internal reason, so this shows up as a countable failure rather than a silent hang. **If a read/write to the session key fails mid-call** (Redis drops during an otherwise-healthy call): treat it the same as any other unrecoverable pipeline error — play the farewell line if there's time, end the call, and let the stuck-call sweep (§10) catch it if even that fails. Log both cases at `error` level; a Redis blip should be visible in monitoring, not silently swallowed.

**Turn-by-turn flow**:
1. Twilio `media` event → forward raw mulaw bytes directly to an open Deepgram WebSocket (`nova-2` model, mulaw/8kHz, `interim_results:true`, `endpointing:300ms`).
2. On any Deepgram result (including interim/non-final) — if the agent is currently speaking or Twilio's playback buffer hasn't drained, **immediately abort the in-flight pipeline and send Twilio a `{event:'clear'}` message**. This is true barge-in — check it on interim results too, not just final ones, so interruption feels instant.
3. Only run the AI pipeline on `speech_final || is_final` with non-empty text. De-dup identical transcripts arriving within ~1.2s (a known STT double-fire quirk).
4. **Core turn logic**:
   - Skip backchannel utterances ("uh", "ok", ≤3 chars) — no LLM call.
   - Every turn increments a turn/pipeline counter; any async step checks it's still the current turn before acting on its result, so a superseded turn (user barged in again) never emits stale audio.
   - Quick regex short-circuit for goodbye-type phrases → skip the LLM, play the farewell line, schedule hangup.
   - Race a knowledge-base search (300ms timeout) against system-prompt assembly, if the utterance looks like a question and the agent has knowledge documents attached.
   - Build the system prompt: substitute contact-name template placeholders, hard-instruct 1–3 sentence voice-appropriate replies (no markdown/lists/emojis), inject hangup instructions (LLM must prepend `[HANGUP] ` when `hangupPrompt` conditions are met) and transfer instructions (`[TRANSFER] ` when `transferConditions` are met). Truncate very long prompts (~3800 chars) for realtime latency.
   - If using function-calling tools (e.g. calendar booking) or a non-streaming-friendly LLM provider, use a non-streaming completion call. Otherwise **stream** the LLM response: buffer tokens until a sentence boundary, fire TTS per-sentence as soon as it's complete (don't wait for the full reply), chain playback so sentences play in order even though TTS generations may finish out of order. Play a "thinking" filler phrase if 1.5s passes with no first sentence yet.
   - After the reply completes: strip `[HANGUP]`/`[TRANSFER]` sentinels, trim to a voice-appropriate length, append to session history (keep last ~6 turns as LLM context) and to `Call.transcriptEntries`.
   - If transfer/hangup was signaled, schedule it after a delay computed from the just-spoken audio's playback duration so the line finishes first.
5. On Twilio `stop`: abort any in-flight pipeline, close the STT socket, delete the session, flatten `transcriptEntries` into `Call.transcript`.

**TTS provider chain**: try Cartesia first (if configured + under a concurrency limit env var), then ElevenLabs, then OpenAI TTS as the always-available fallback. Chunk audio into 20ms/160-byte frames for Twilio streaming.

## Gather fallback pipeline (Deepgram not configured)

Turn-based: Twilio POSTs `SpeechResult` → same system-prompt assembly (not raced/parallelized, this path is already slower) → non-streaming LLM reply → TwiML `<Say>`/`<Play>` + next `<Gather>` or `<Dial>`/`<Hangup>`. Store conversation history in Redis keyed by `callSid` (not a local Map — same reasoning as the streaming path).

## Human handoff (live transfer to a person)

This is the mechanism behind the `transferEnabled`/`transferPhoneNumber`/`transferConditions` fields on the Agent (Prompt 02) — build it as a first-class capability, not an afterthought:

1. The system prompt (per the turn logic above) instructs the LLM to prepend `[TRANSFER] ` to its reply the moment `transferConditions` are met (e.g. the caller explicitly asks for a person, or the conversation matches an urgent/escalation pattern the owner described in plain English).
2. When `[TRANSFER]` is detected: speak the agent's `transferMessage` ("One moment, connecting you now...") in full, then — once that audio has finished playing (compute the delay from the audio's own duration, don't guess a fixed pause) — issue a Twilio `<Dial>` to `transferPhoneNumber`, with `callerId` set to the Twilio-owned number handling this call (see the session-state note above; Twilio rejects a `<Dial>` whose `callerId` isn't owned by the account placing it).
3. Set `Call.transferred = true` the moment the `<Dial>` actually fires — this is a real signal for reporting ("how many calls needed a human"), not just a conversational aside, so don't skip persisting it.
4. If `transferPhoneNumber` doesn't answer, fail gracefully — TwiML's own `<Dial>` timeout/fallback (e.g. an apology + normal hangup) rather than leaving the caller in dead air.
5. `POST /:id/test-transfer` (Prompt 02) exercises this exact path outside of a real conversation, so an owner can verify their transfer number actually rings before relying on it live.

## Appointment booking (Cal.com function-calling)

Only enable this for an agent with `calcomEventTypeId` set **and** the owner having a `connected` Cal.com `ProviderConnection` (Prompt 08) — resolve both before offering booking at all; if either is missing, the agent should never claim it can book anything.

1. **Tool definitions**: expose two function-calling tools to the LLM — `get_calendar_availability(date_range)` and `book_appointment(start_time, attendee_name, attendee_phone, attendee_email?)`. Only wire this up for LLM providers/models that actually support function-calling (verify per-provider — don't silently no-op it for a model that doesn't support tools, disable the capability instead so the agent doesn't reference booking it can't perform).
2. **Non-streaming path required**: per the turn logic above, a turn that may invoke a tool must use a non-streaming completion call, since you need the full tool-call payload before acting — don't try to stream through a function-calling turn.
3. **Availability check**: on `get_calendar_availability`, call Cal.com's availability API for the agent's `calcomEventTypeId` in `calcomTimeZone`, and return the open slots as the tool result so the LLM can offer real times to the caller (never let the LLM invent a time slot itself).
4. **Booking**: on `book_appointment`, call Cal.com's booking-creation API with the confirmed slot and attendee details. **Only on a genuine success response**: set `Call.appointmentBooked = true` and `Call.appointmentAt` to the booked start time, and let the LLM confirm verbally ("You're booked for Friday at 3 PM — you'll get a confirmation email"). On any failure (no availability, API error, expired slot): the tool result fed back to the LLM must clearly say it failed, so the agent tells the caller honestly and offers a fallback (e.g. "let me have someone follow up to confirm a time") — **never let the agent claim a booking succeeded when the API call actually failed.**
5. Store the raw Cal.com booking reference (e.g. a booking UID) in `Call.extractedData` or a dedicated field if you want to support later cancellation/rescheduling — optional, but don't discard it if you have it.

## Inbound & browser-based calls

**Inbound**: look up the `PhoneNumber` by the dialed number; hang up if not found, if it's a shared/platform number (shared numbers are outbound-only), or if no inbound agent is assigned. Find-or-create a `Contact` by phone. Create the `Call` at `status:'in-progress'` (inbound calls are connected immediately, unlike outbound's `queued→ringing`). Register status/recording callbacks using **this org's own Twilio credentials**. Fire an `inbound_call` workflow trigger (Prompt 06).

**Browser-based (WebRTC) calling** is optional — only build it if an in-app "call from the browser" feature is wanted; the public embeddable widget itself is out of scope per the master guide.

## Call completion & duration resolution

Three independent paths can mark a call complete — make sure none of them double-process:

1. **Status webhook**: map Twilio's status vocabulary to your internal enum; **never downgrade an already-terminal status**. On voicemail-detection (AMD) results, force `status:'voicemail'`. On completion, resolve duration: Twilio's `CallDuration` → last-transcript-timestamp estimate (+30s buffer) → full wall-clock elapsed, always capped at the agent's `maxDurationSeconds`. Use an atomic `findOneAndUpdate` with a `workflowTriggered:{$ne:true}` filter to claim the "fire `call_completed` trigger" slot exactly once. Kick off post-call processing (fire-and-forget).
2. **Recording webhook** (arrives after the call ends, sometimes much later): store the recording URL; if Twilio gave an exact duration, it overrides any estimate. **Safety net**: if the call is still `in-progress` at this point (status webhook was lost), complete it here too, using the same duration-resolution chain and the same atomic trigger-claim pattern. Always re-run post-call processing (a late recording can enable a transcription fallback that wasn't possible before).
3. **Manual completion** (`POST /api/calls/:id/complete`, user-facing "mark this done" action for a stuck call): skip processing if the call never reached `in-progress` (no talk time). Otherwise estimate duration from `startedAt`→now, capped, and run post-call processing.
4. **Stuck-call cleanup sweep** (Prompt 11, every 2 minutes): force-completes any call stuck `in-progress` for >15 minutes using the same estimate chain. Keep this — it's a correctness safety-net independent of billing.

## Post-call processing

Run (non-blocking) if the agent's task settings have anything enabled:
- Idempotency guard: skip if already fully processed **and** a transcript exists, unless the transcript was previously empty (a late recording just made transcription possible).
- If there's no live transcript but a recording exists, transcribe it with Whisper.
- Run knowledge-gap detection independently of task-setting toggles (gated only by a user preference, default on — see Prompt 07).
- If enabled, run **in parallel**: summarization (free-text GPT call), structured-data extraction (JSON-mode GPT call using the agent's custom extraction prompt; derive `sentiment` from the result), success-score evaluation (JSON-mode GPT call, 0–100, using a custom or default rubric). Each independently try/catch'd.
- If a webhook URL is configured, POST the results (callId, from/to, status, duration, summary, extractedData, callScore, sentiment, transcript) with a 10s timeout; log failure, don't retry.

## Environment variables
`TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_WEBHOOK_BASE_URL` (public HTTPS base URL Twilio calls back to), `TWILIO_PHONE_NUMBER` (default caller ID), `DEEPGRAM_API_KEY` (unset → Gather fallback), `OPENAI_API_KEY` (LLM + TTS fallback + Whisper), `ANTHROPIC_API_KEY` (if used), `CARTESIA_API_KEY` + `CARTESIA_CONCURRENT_LIMIT`, `ELEVENLABS_API_KEY`. Cal.com credentials are **not** a global env var — the appointment-booking flow above resolves them per-owner from `ProviderConnection` (Prompt 08).

## Dependencies
`twilio` SDK, `ws`, `openai` SDK, `@anthropic-ai/sdk` (if kept). Deepgram/Cartesia/ElevenLabs via raw WebSocket/`fetch` — no SDK needed.

## Build this multi-instance-ready from the start (100k+/day, 1,000+ concurrent)

This is not an optimization to do later — build it this way the first time:
- **Session state → Redis**, not an in-process `Map`, for both the streaming and Gather-fallback pipelines. At 1,000+ concurrent calls you need either shared state or sticky-routing-by-`callId` at the load balancer; plan on Redis-backed state as the primary design.
- **TTS audio files → S3 or regenerate-on-demand**, never local disk assumed to be served by the same instance.
- **Size Cartesia's concurrency limit and the Twilio/Deepgram account plans for 1,000+ concurrent** — these are provider-side budgeting items, flag them explicitly rather than silently hitting a ceiling in production.
- **Mongo write throughput**: every conversation turn writes a transcript entry; size your connection pool and consider Atlas (or a properly sized cluster) for this write volume.
- Plan for the API process to run as multiple horizontally-scaled instances behind a load balancer from the outset, with a separate worker-role process for background jobs (Prompt 09) — don't build this as a single all-in-one process and hope it scales later.

## Acceptance checklist
- [ ] An outbound call places successfully, greets the caller, and holds a multi-turn conversation with sub-1-second response latency in streaming mode.
- [ ] Barge-in works: speaking while the agent is talking immediately stops its audio.
- [ ] An inbound call to an owned number with an assigned agent connects and behaves identically to an outbound call from the AI's side.
- [ ] A call's `durationSeconds` and `transcript` are correctly populated regardless of whether the status webhook, recording webhook, or manual-complete path is what actually finished it — and it's never processed twice, including every side effect (notification, workflow trigger, webhook delivery), not just the status change itself.
- [ ] Manually replaying the same Twilio status webhook payload twice in a row (simulating their automatic retry behavior) produces no duplicate notification, no duplicate workflow trigger, and no duplicate outbound webhook delivery.
- [ ] A call's detail page still shows a sensible agent name after that agent has been deleted.
- [ ] Session state for an in-flight call is recoverable from Redis, not lost if the handling instance restarts (or the deployment routes subsequent messages to the same instance via sticky routing — document which approach you took).
- [ ] Twilio webhook routes reject requests without a valid `X-Twilio-Signature`.
- [ ] A conversation matching an agent's `transferConditions` finishes the `transferMessage` audio, then bridges to `transferPhoneNumber` with a valid Twilio-owned `callerId`, and `Call.transferred` is set to `true`.
- [ ] For an agent with Cal.com connected, a caller can hear real availability and get a real appointment booked mid-call, with `Call.appointmentBooked`/`appointmentAt` set only from a genuine Cal.com API success — never optimistically.
- [ ] If Cal.com is disconnected, unreachable, or returns an error, the agent tells the caller honestly instead of claiming a booking succeeded.
