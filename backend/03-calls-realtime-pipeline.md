# 03 — Calls & the Real-Time Voice Pipeline

> Backend domain doc. Source: `backend/src/{models/Call.model.ts, controllers/call.controller.ts, routes/call.routes.ts, services/call.service.ts, controllers/twilio.controller.ts, routes/twilio.routes.ts, services/twilio.service.ts, handlers/mediaStream.handler.ts, services/ttsAudio.service.ts, services/postCall.service.ts, validations/call.validation.ts}`.

## 1. Overview

This is the most complex domain in the system: every phone call (manual outbound, inbound, campaign-dialed, or workflow-automation) is represented by one `Call` document and driven through Twilio, with the actual AI conversation happening either over a **real-time WebSocket media stream** (Deepgram STT + streaming LLM + low-latency TTS) or, when Deepgram isn't configured, a **classic turn-based `<Gather>` loop** (Twilio's own STT). Both paths converge on the same `Call` record and the same post-call pipeline (billing, summarization, workflow triggers).

There are **four ways a call gets created**:
1. **Manual dispatch** — `POST /api/calls` (agent detail page "test call" / quick-dial), handled by `call.service.dispatchCall`.
2. **Inbound** — someone dials a Twilio number the platform owns; `twilio.controller.inboundCall` → `twilio.service.handleInboundCall`.
3. **Campaign dialer** — `campaign.worker.ts` calls `initiateOutboundCall` directly for each contact in a running campaign (see `04-campaigns.md`).
4. **Workflow automation follow-up** — `workflowFollowup.worker.ts` calls `call.service.dispatchCall(..., workflowFollowup: true)` (see `06-workflows.md`).

## 2. Data Model (`models/Call.model.ts`)

| Field | Type | Notes |
|---|---|---|
| `agentId` | ObjectId → Agent | required |
| `userId` | ObjectId → User | required — the owner, used for every ownership/billing check |
| `contactId` | ObjectId → Contact | optional |
| `campaignId` / `campaignContactId` | ObjectId | set only for campaign-dialed calls |
| `status` | enum | `queued \| ringing \| in-progress \| completed \| failed \| no-answer \| busy \| voicemail` |
| `direction` | enum | `inbound \| outbound` |
| `from` / `to` | string | E.164 |
| `twilioCallSid` | string | filled once Twilio accepts the call |
| `recordingUrl` / `recordingSid` | string | filled by the recording webhook, arrives *after* the call ends |
| `durationSeconds` | number | see §7 for the multi-source resolution logic — this is not a simple "one webhook sets it" field |
| `transcript` | string | flattened `"Agent: ...\nUser: ..."` text, built when the call ends |
| `transcriptEntries` | `{role, text, timestamp}[]` | structured, appended turn-by-turn *during* the call |
| `sentiment` | enum | `positive\|neutral\|negative`, set by post-call extraction |
| `callScore` | number | 0–100, set by post-call success evaluation |
| `greetingAudioUrl` | string | pre-generated greeting MP3 URL for non-Polly voices |
| `summary` / `extractedData` | string / Mixed | post-call GPT output |
| `webhookSent` | boolean | whether the agent's post-call webhook was delivered |
| `workflowFollowup` | boolean | true if this call **was itself** an automation follow-up call — prevents a follow-up call from triggering another follow-up (infinite loop guard) |
| `workflowTriggered` | boolean | atomic one-shot flag — set the instant the `call_completed` workflow trigger fires for this call, so two racing webhooks (status callback vs. recording callback) can't double-fire it |
| `startedAt` / `completedAt` | Date | |

### Indexes
```js
{ agentId: 1, createdAt: -1 }
{ userId: 1, createdAt: -1 }
{ agentId: 1, status: 1, createdAt: -1 }
{ campaignId: 1, createdAt: -1 }
{ twilioCallSid: 1 }  // sparse
{ status: 1, createdAt: 1 }  // partial: status:'in-progress' only — powers the stuck-call cleanup sweep (see 11-app-bootstrap-infra.md)
```

## 3. Dispatching a call — `call.service.dispatchCall`

```
dispatchCall(userId, agentId, to, direction, contactId?, workflowFollowup?)
```
1. Loads the `Agent`, checks ownership (`403`) and `status !== 'inactive'` (`403`).
2. **Quota gate**: `assertCanPlaceVoiceCall(userId, direction)` — throws `402` if the account can't place calls (billing/plan-driven; see §11 for rebuild guidance).
3. **Caller-ID resolution** — two different rulesets:
   - **Automation calls** (`workflowFollowup: true`) *never* fall back to the platform's shared Twilio number: agent's `callSettings.phoneNumberId` → owner's default active non-company `PhoneNumber` → any active non-company number owned by the user. If none exists, throws `400 WORKFLOW_CALLER_ID` with a message telling the user to add a number — this is a deliberate policy so automated calls always go out under the tenant's own identity.
   - **Manual/ad-hoc calls** — `TWILIO_PHONE_NUMBER` env (platform default) → agent's `phoneNumberId` → owner's default active number. If Twilio itself isn't configured at all, no caller-ID error is thrown (falls through to the dev-simulation path).
4. Creates the `Call` row (`status:'queued'`), bumps `Contact.totalCalls`/`lastCalledAt` if linked.
5. **If Twilio is configured** (`TWILIO_ACCOUNT_SID` + `TWILIO_AUTH_TOKEN` + `TWILIO_WEBHOOK_BASE_URL` all set) → `initiateOutboundCall(...)` places a real call.
6. **Else** → enqueues a `process-call` job on the BullMQ `call-processing` queue, which the worker (`call.worker.ts`) simulates with a fake delay + transcript and marks completed — this is a **local-dev-without-Twilio-credentials** fallback path, not used in any real deployment.

### Validation (`validations/call.validation.ts`)
`dispatchCallSchema` accepts either `phoneNumber` or the legacy `to` field (normalizes `-`, spaces, parens out, prefers `phoneNumber`), requires E.164-ish format (`/^\+?[1-9]\d{6,14}$/`), defaults `direction` to `'outbound'`.

## 4. API Reference

### `/api/calls` (also mounted at `/api/agents/:agentId/calls`) — JWT required

| Method | Path | Rate limit | Purpose |
|---|---|---|---|
| POST | `/` | 20/min/IP (`callDispatchLimiter`) | Dispatch a new call |
| GET | `/` | — | List the caller's calls (or, when nested under `/agents/:agentId/calls`, only that agent's calls) — paginated, `contactId`/`agentId` populated |
| GET | `/:id` | — | Single call detail (populates `agentId`, `contactId`) |
| POST | `/:id/complete` | — | **Manual completion** — see §8 |

### `/api/twilio` — all PUBLIC (no JWT; Twilio calls these directly)

| Method | Path | Purpose |
|---|---|---|
| GET/POST | `/voice/:callId` | Initial TwiML when the call connects |
| GET | `/greeting-audio/:callId` | Serves pre-generated greeting MP3 (for Cartesia/ElevenLabs in stream mode) |
| POST | `/gather/:callId` | Turn-based conversation loop (non-Deepgram fallback) |
| POST | `/status/:callId` | Call status changes (ringing→in-progress→completed etc.) |
| POST | `/recording/:callId` | Recording ready (arrives after call ends) |
| POST | `/inbound/:phoneNumberId` | Inbound call to a specific owned number |
| POST | `/inbound` | Generic/unrecognized inbound number fallback — plays a static message and hangs up |
| POST | `/widget/voice-twiml` | TwiML App voice URL for browser-widget calls |
| GET | `/recording-proxy?url=` | Proxies a Twilio recording (Basic-Auth-protected) to the frontend, injecting credentials server-side, following redirects up to 5 hops |

**Every handler always returns HTTP 200 with valid TwiML**, even on internal error — Twilio treats any non-200 or malformed-XML response as an "application error" and plays a generic message; the codebase instead defines its own `VOICE_FALLBACK_TWIML`/`HANGUP_TWIML` constants so failures are graceful and diagnosable from logs rather than a mystery Twilio error tone.

## 5. Initial TwiML — `buildInitialTwiml` (`twilio.service.ts:293`)

Called for every new call (inbound, outbound, widget) once Twilio requests `/voice/:callId`:

1. Loads `Call` + populated `Agent`.
2. Resolves the greeting line (agent's `callSettings.greetingMessage` or a default `"Hello, I'm {agent.name}..."`).
3. Resolves which `<Say voice="...">` Polly voice to use via `resolveTwilioVoice()` — maps OpenAI voice names to the *nearest-sounding* Twilio-native Neural Polly voice (§6) so a plain `<Say>` can be used even for agents configured with `openai` as their voice provider, when native audio isn't otherwise needed.
4. If the agent's provider is `cartesia` or `elevenlabs` (**needs pre-generated audio** — Twilio's `<Say>` only supports Polly natively) **and** Deepgram isn't configured, the greeting is synthesized up front and served via `<Play>` from a temp file. In stream mode (Deepgram configured), the greeting is instead fetched via the `/greeting-audio/:callId` **route** (not inline) so any instance behind a load balancer can serve it without needing the exact file on local disk.
5. **Branches on `DEEPGRAM_API_KEY`**:
   - **Set** → returns TwiML with `<Connect><Stream url="wss://.../media-stream/:callId"/></Connect>`, followed by a **silent Gather fallback** and `<Hangup>` (in case the media stream exits early/never connects, the call doesn't just go dead). The stream URL is **path-based** (`/media-stream/<callId>`, not a query string) specifically because reverse proxies (Nginx) commonly strip query strings from WebSocket upgrade requests — a bug class already hit and worked around.
   - **Not set** → classic `<Gather input="speech" action=".../gather/:callId" timeout="5" speechTimeout="auto">` loop.

## 6. Voice provider → Twilio mapping

```
OpenAI voice → nearest Twilio Polly Neural voice (used for <Say>, free with Twilio):
  alloy → Polly.Matthew-Neural   echo → Polly.Stephen-Neural
  fable → Polly.Brian-Neural     onyx → Polly.Gregory-Neural
  nova  → Polly.Joanna-Neural    shimmer → Polly.Ruth-Neural
```
`twilio-polly` provider uses the configured `voiceId` directly (already a `Polly.X` string). `cartesia`/`elevenlabs` always need pre-generated/streamed audio — they cannot use `<Say>` at all.

## 7. Path A — Real-time Media Stream pipeline (`handlers/mediaStream.handler.ts`)

Active only when `DEEPGRAM_API_KEY` is set (`attachMediaStreamServer` no-ops otherwise, logging that the Gather fallback is active). This is where the product's actual "feels like a real phone conversation" quality comes from — sub-1-second turn latency with true barge-in.

### Wiring
`attachMediaStreamServer(server)` (called once from `index.ts`) hooks the raw Node `http.Server`'s `'upgrade'` event: any request whose URL starts with `/media-stream` is manually completed via `wss.handleUpgrade(...)`, running a `WebSocketServer` **on the same port** as Express rather than a separate server/port.

### Connection setup (race-condition handling)
Twilio can send the `start` event within milliseconds of the socket opening — **before** the handler's async `Call.findById(...).populate('agentId')` DB lookup resolves. To avoid losing it: all incoming messages are buffered into an array via a temporary listener the instant the WS connects; once the DB lookup (and Cal.com context resolution — wrapped in try/catch so a Cal.com API hiccup can't kill the whole session) completes, the real message handler is installed and every buffered message is replayed in order.

`callId` is parsed from the path (`/media-stream/<callId>`), with a legacy query-string (`?callId=`) fallback still supported. Invalid/missing `callId` (or a call not found in Mongo) closes the socket immediately.

### Session state (per-call, in-memory, keyed by `callId` in a `Map`)
Tracks: WS handles (Twilio + Deepgram), conversation `history`, the loaded `agent` config, resolved Cal.com tool context, `contactName` (for prompt templating), `isSpeaking`/`isSpeakingUntil` (drives barge-in), an `AbortController` + incrementing `pipelineId` (lets a new turn cleanly cancel/ignore a stale in-flight turn), and `transferCallerId` (which Twilio-owned number to present as caller ID on a `<Dial>` transfer — `call.to` for inbound, `call.from` for outbound, since Twilio requires the callerId on a `<Dial>` to be a number owned by the account placing the bridge).

### Turn-by-turn flow
1. **Twilio `media` event** → raw base64 mulaw audio bytes are forwarded directly to an open **Deepgram** WebSocket (`nova-2` model, `mulaw`/8kHz, `interim_results:true`, `endpointing:300`ms).
2. **Deepgram `Results` message** → on every message, if speech is detected **while the agent is currently speaking or Twilio's playback buffer hasn't drained yet** (`isSpeaking || Date.now() < isSpeakingUntil`), immediately: abort the in-flight pipeline, send Twilio a `{event:'clear'}` message (flushes any queued audio Twilio hasn't played yet), and reset speaking state — this is **true barge-in**, checked on interim (non-final) results too, so the interruption feels instant rather than waiting for the user's utterance to fully finish.
3. Only on `speech_final || is_final` **with non-empty text** does `handleTranscript()` actually run the AI pipeline. A de-dup guard drops an identical transcript repeated within 1200ms (a known Deepgram double-fire quirk).
4. **`handleTranscript(session, userText)`** (the core pipeline, target <1s to first spoken word):
   - Ignores **backchannel utterances** ("uh", "ok", "yeah", ≤3 chars, etc.) — no LLM call, no response.
   - Ignores **stale sessions** — every turn increments `session.pipelineId`; any async step later in the pipeline checks `myPipelineId === session.pipelineId` before acting, so a superseded turn (because the user barged in again) never writes duplicate/late audio.
   - **Quick farewell short-circuit**: a regex match on goodbye-ish phrases skips the LLM entirely and plays the farewell line, then schedules hangup.
   - **Parallel KB search**: only triggered if the utterance looks like a question (has `?` or interrogative/domain keywords) and is ≥20 chars and the agent has `knowledgeDocIds`; raced against a 300ms timeout so a slow vector search never blocks the turn — whatever KB context arrives in time gets appended to the system prompt.
   - **System prompt assembly** happens concurrently with the KB race: injects contact-name template substitution (`{prospect_name}`, `{contact_name}`, `{first_name}`, `{name}`), a hard instruction to keep responses to 1–3 sentences with no markdown/lists/emojis (this is a voice call), hangup instructions (if `hangupPrompt` set → LLM must prepend `[HANGUP] ` when done), and transfer instructions (if `transferEnabled` → LLM must prepend `[TRANSFER] ` when the configured conditions are met). Long prompts are truncated to ~3800 chars for realtime mode latency (`toRealtimeVoicePrompt`).
   - **Branch: Cal.com tools or Anthropic provider** → uses the **non-streaming** `generateWidgetReply()` (function-calling/tool-use doesn't support token streaming in this implementation; Anthropic is also routed non-streaming for simplicity) — reply arrives whole, then goes straight to TTS.
   - **Branch: normal OpenAI streaming** → opens a streamed `chat.completions.create(..., stream:true)` call. As tokens arrive, they're buffered until a sentence boundary (`/[.!?]+\s+/` regex, minimum 8 chars) is detected; each complete sentence is immediately hand off to `enqueueSentence()`, which **starts TTS synthesis right away** (non-blocking) while continuing to consume more LLM tokens. A **playback chain** (chained promises) guarantees sentences are sent to Twilio **in order** even though their TTS generations may finish out of order/concurrently. A **"thinking" filler** (`callSettings.thinkingMessage`) plays automatically if 1.5s passes with no first sentence yet, canceled the instant real audio is ready.
   - After the full LLM reply is collected, `[HANGUP]`/`[TRANSFER]` sentinels are parsed off the front, the reply is trimmed (`compactVoiceReply` — caps at 2 sentences/220 chars normally, 4 sentences/420 chars if it looks like a booking-confirmation reply), and the turn is appended to `session.history` (last 6 turns kept as LLM context) and to `Call.transcriptEntries` in Mongo.
   - If a transfer/hangup was signaled, `scheduleCallTransfer`/`scheduleCallHangup` fire on a delay calculated from the just-spoken audio's playback duration (so the line finishes before the call is redirected/ended).
5. **On Twilio `stop` event**: aborts any in-flight pipeline, closes the Deepgram socket, deletes the in-memory session, and flattens `transcriptEntries` into the flat `Call.transcript` string.

### TTS provider chain (`ttsToMulaw`, mediaStream.handler.ts)
1. **Cartesia** — if `CARTESIA_API_KEY` set, agent's voice provider is `cartesia`, and under the configurable concurrency limit (`CARTESIA_CONCURRENT_LIMIT` env, default 12 — tied to the Cartesia plan's concurrent-call quota). Requests raw PCM at 8kHz directly (no downsampling needed), then encodes to μ-law.
2. **ElevenLabs** — if configured and agent uses `elevenlabs`; requests `ulaw_8000` output format directly (ElevenLabs natively outputs exactly what Twilio needs), model `eleven_turbo_v2_5` for low latency.
3. **OpenAI** (always-available fallback) — `tts-1` model, PCM output, then manually downsampled 24kHz→8kHz (simple 3-sample averaging, `downsample24kTo8k`) and encoded to μ-law (`linearToMuLaw`/`pcm16ToMuLaw` — a textbook G.711 μ-law encoder implemented inline).

Audio is chunked into 160-byte (20ms @ 8kHz) frames and sent to Twilio as base64-encoded `media` WebSocket messages (`streamAudioToTwilio`), which also updates `session.isSpeakingUntil` (audio length in ms + 300ms network-buffer padding) — this is the value barge-in detection checks against.

## 8. Path B — Classic Gather pipeline (`twilio.service.handleGather`)

Used whenever `DEEPGRAM_API_KEY` is not set. Turn-based, driven entirely by Twilio's own speech recognition:
1. Twilio POSTs `SpeechResult` to `/api/twilio/gather/:callId`.
2. An **in-memory `Map<callSid, history[]>`** (`callSessions`, module-level — **not** Redis, so it doesn't survive a process restart or work across multiple instances) holds conversation history per call.
3. Same system-prompt assembly as the streaming path (KB search, hangup/transfer instructions, Cal.com context) but **not parallelized/raced** — this path is inherently already several seconds slower per turn (Twilio STT round-trip + non-streaming LLM call), so the same latency optimizations weren't applied.
4. `generateWidgetReply()` (non-streaming) gets the full reply, `[HANGUP]`/`[TRANSFER]` sentinels are parsed the same way, transcript entries are saved.
5. Returns TwiML: `<Say>`/`<Play>` (Cartesia/ElevenLabs need `<Play>` via a pre-generated file URL) + either `<Dial>` (transfer), `<Hangup>` (hangup), or another `<Gather>` for the next turn.
6. A recognized end-phrase regex (`goodbye|bye|hang up|...`) short-circuits straight to the farewell + hangup without an LLM call, same as the streaming path.

## 9. Inbound & Widget call entry points

**`handleInboundCall(phoneNumberId, from, callSid)`**:
- 404-equivalent hangup if the `PhoneNumber` doesn't exist, or if it's flagged `isCompany` (platform-pool/shared numbers are outbound-caller-ID-only and never accept AI inbound), or if it has no `inboundAgentId` configured.
- Quota-gates via `canPlaceVoiceCall` — on failure, plays a deliberately **vague, non-account-specific** message (`callerFacingQuotaBlockMessage`) so a random inbound PSTN caller never learns anything about the business's billing state.
- Finds-or-creates a `Contact` by phone number, creates the `Call` (`status:'in-progress'` — inbound calls are considered connected immediately, unlike outbound which starts `queued`→`ringing`).
- Registers status/recording callbacks using the **owner's own Twilio client** (`getUserTwilio`, from `phoneNumber.service.ts`) — critical: the call is happening on the *user's* Twilio account/number, not the platform's, so platform credentials can't be used to modify it.
- Fires an `inbound_call` workflow trigger (see `06-workflows.md`).

**`handleWidgetCall(agentId, callSid)`** — near-identical but for browser-based (Twilio Voice SDK / WebRTC) calls initiated from the embeddable widget: `from:'Browser Widget'`, `to:'AI Agent'`, no phone number involved. (Public widget is out of scope for the in-house rebuild per current project scoping — documented here only because it shares the same TwiML/pipeline machinery as phone calls.)

## 10. Call completion, duration resolution & billing trigger

Three independent code paths can mark a call `completed` and must not double-process it — this is one of the gnarlier parts of the codebase to replicate faithfully:

### `handleStatusCallback` (Twilio status webhook)
- Maps Twilio's status vocabulary to the internal enum (`answered`→`in-progress`, `canceled`→`failed`, etc.) and **never downgrades a status already terminal** (`completed|failed|no-answer|busy|voicemail`) — guards against a late/out-of-order `ringing` callback overwriting a call that's already finished.
- On `AnsweredBy === 'machine_start'|'machine_end_beep'` (Twilio AMD), forces `status:'voicemail'`.
- On transition into `in-progress`/`answered` for the **first time** on a non-followup call, fires the `call_started` workflow trigger.
- On `completed`: resolves `durationSeconds` with a priority chain — **Twilio's `CallDuration`** first, else **last transcript-entry timestamp minus `startedAt` + 30s buffer** (best no-recording estimate), else **full wall-clock elapsed** (least accurate, still capped) — always clamped to the agent's `maxDurationSeconds`. Then calls `recordVoiceCallBillableMinutes(userId, minutes)` (billing domain — see rebuild note below) and creates a user-facing `Notification` summarizing minutes used/charged.
- **Double-billing guard**: only bills if `durationSeconds` wasn't already set by an earlier path (the recording callback can race ahead of the status callback).
- **Workflow trigger race guard**: uses an atomic `findOneAndUpdate({_id, workflowTriggered:{$ne:true}}, {$set:{workflowTriggered:true}})` to claim the "fire `call_completed`" slot exactly once, regardless of which webhook gets there first.
- Kicks off `processPostCall(callId)` (non-blocking, fire-and-forget with its own error logging).

### `handleRecordingCallback` (Twilio recording webhook — arrives *after* the call ends, sometimes well after)
- Stores `recordingUrl`/`recordingSid`, and if Twilio supplied an exact `recordingDuration`, uses it immediately as the authoritative duration (overrides any estimate).
- **Safety-net completion**: if the call is *still* `in-progress` at this point (meaning the status callback never fired, was lost, or hasn't arrived yet), this handler independently completes and bills the call using the same duration-priority chain as above (`recordingDuration` → already-stored → transcript-estimate → `maxDurationSeconds`), and independently claims the workflow-trigger slot via the same atomic pattern.
- Always re-runs `processPostCall` afterward, since a recording arriving late means a recording-based Whisper transcription fallback may now be possible where none was before.

### `POST /api/calls/:id/complete` — manual completion (`call.controller.manualComplete`)
A user-facing "give up waiting, mark this done" action for a call stuck in `ringing`/`in-progress` from the UI. Skips billing entirely if the call never reached `in-progress` (no talk time = nothing to bill). Otherwise estimates duration from `startedAt`→now (capped at the agent's max), bills, and runs `processPostCall`.

There is also a **fourth**, fully independent safety net: a **stuck-call cleanup sweep** in `index.ts` (every 2 minutes) that finds calls stuck `in-progress` for >15 minutes and force-completes+bills them using the same estimate chain — documented in `11-app-bootstrap-infra.md`.

## 11. Post-Call Processing (`postCall.service.ts`)

Runs (non-blocking, called from every completion path above) if the `Agent.taskSettings` has anything enabled:
- **Idempotency guard**: skips if the call already has `summary` + `callScore` + `extractedData` **and** a transcript exists — unless the transcript was previously empty (i.e., a late-arriving recording just made transcription possible for the first time), in which case it's allowed to re-run.
- **Recording-based transcription fallback**: if there's no live transcript (e.g. the media-stream/Gather turn logging failed or was skipped) but a `recordingUrl` exists, downloads the recording (Basic-Auth with Twilio credentials) and runs OpenAI Whisper (`whisper-1`) to produce one.
- **Knowledge-gap detection** runs independently of the task-settings toggles, gated only by `UserSettings.knowledgeGapAnalysisEnabled` (default true) — see `07-knowledge-base.md`.
- If **summarization**, **extraction**, and **success evaluation** are each enabled, they run **in parallel** (`Promise.all`) against the same transcript+system-prompt context block, each independently try/catch'd so one failing task doesn't block the others:
  - Summarization → free-text GPT call (`gpt-4o-mini`), 3–5 bullet points.
  - Extraction → GPT call forced to `response_format: json_object`, using the agent's custom `extractionPrompt`; also derives `sentiment` from the extracted data's `sentiment` field if present.
  - Success evaluation → GPT call forced to JSON, using the account's custom `aiSuccessScorePrompt` (from `UserSettings`) or a default rubric; score clamped 0–100.
- **Webhook delivery**: if `taskSettings.webhookUrl` is set, POSTs a JSON payload (callId, agentId, from/to, status, duration, summary, extractedData, callScore, sentiment, transcript) with a 10s timeout; sets `webhookSent:true` on success. Failure is logged, not retried.

## 12. Rebuild Notes (Replit target, in-house/no-billing)

### Billing coupling to remove or stub
Nearly every completion path calls `recordVoiceCallBillableMinutes` (billing domain, explicitly out of scope for this rebuild) and `assertCanPlaceVoiceCall`/`canPlaceVoiceCall` (usage-quota, also billing-driven). For an in-house build with no metered billing:
- Replace `assertCanPlaceVoiceCall(userId, direction)` in `call.service.dispatchCall` with a no-op (or a simple "is this account active/not suspended" check using just the `User` model).
- Replace `canPlaceVoiceCall(...)` calls in `twilio.service.handleInboundCall`/`handleWidgetCall` similarly.
- Remove the `recordVoiceCallBillableMinutes(...)` calls in `handleStatusCallback`, `handleRecordingCallback`, `call.controller.manualComplete`, and the stuck-call cleanup job (`index.ts`) — along with the billing-derived user-facing `Notification` text ("X minutes used, $Y charged"), which should be replaced with a plain "Call completed, N seconds" notification if you want to keep the notification at all.
- `callerFacingQuotaBlockMessage` becomes unnecessary once the quota gate is removed.

### Dependencies
`twilio` SDK, `ws` (WebSocket server), `openai` SDK (LLM + Whisper + TTS), `@anthropic-ai/sdk` (if Anthropic LLM support is kept), Deepgram is accessed via raw WebSocket (no SDK dependency, just `wss://api.deepgram.com/v1/listen`), Cartesia/ElevenLabs via raw `fetch`/`https` calls (no SDK).

### Env vars required for this domain
| Variable | Effect if unset |
|---|---|
| `TWILIO_ACCOUNT_SID` / `TWILIO_AUTH_TOKEN` | No real calls possible — falls back to BullMQ dev simulation for dispatch; inbound/webhook paths simply can't function |
| `TWILIO_WEBHOOK_BASE_URL` | Must be a publicly reachable HTTPS URL (ngrok in local dev, real domain in prod) — Twilio calls back to this for TwiML/status/recording |
| `TWILIO_PHONE_NUMBER` | Platform-default outbound caller ID when no other number resolves (manual calls only — automation calls never use this) |
| `DEEPGRAM_API_KEY` | Unset → falls back entirely to the classic `<Gather>` turn-based pipeline (still fully functional, just higher latency and no true barge-in) |
| `OPENAI_API_KEY` | Required for: OpenAI LLM, OpenAI TTS fallback (always-available tier), Whisper post-call transcription |
| `ANTHROPIC_API_KEY` | Required only if any agent is configured with `llm.provider:'anthropic'` |
| `CARTESIA_API_KEY` / `CARTESIA_CONCURRENT_LIMIT` | Unset → Cartesia voice agents silently fall back to OpenAI TTS |
| `ELEVENLABS_API_KEY` | Unset → ElevenLabs voice agents fall back to OpenAI TTS |
| `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY`/`AWS_REGION` | Only used for Polly voice **preview** elsewhere (voices page) — the actual `<Say voice="Polly.X-Neural">` TwiML doesn't need AWS credentials since Twilio itself renders Polly voices |

### Storage note
`ttsAudio.service.ts` writes pre-generated greeting/reply MP3s to **local disk** (`/tmp/tts-audio/`), auto-purged after 30 minutes, served via `/audio/:file`. On Replit, `/tmp` is ephemeral per-instance/restart, which is fine given the 30-minute self-purge — but if the deployment ever runs **multiple concurrent instances**, an instance that generated a file might not be the one Twilio's `<Play>` request reaches. The stream-mode greeting path already routes around this (fetches via `/greeting-audio/:callId`, which regenerates from the DB+agent config on demand rather than reading a pre-written file) — the same pattern (serve-by-regenerating, not serve-by-reading-a-known-path) should be used everywhere if the in-house rebuild ever scales beyond one instance.

### In-memory state that won't survive a restart or multi-instance deployment
- `callSessions` (Gather-mode conversation history, `twilio.service.ts`)
- `sessions` (media-stream session state, `mediaStream.handler.ts`)
- `cartesiaActiveCalls` counter
- `verifiedTwiMLApps` cache

All are acceptable for a single-instance deployment (a mid-call process restart already drops the call from Twilio's perspective too, so losing in-memory state alongside it isn't a new failure mode) — just don't assume any of this state is shared if you ever run 2+ instances behind a load balancer without sticky sessions.

### Things to verify/decide before rebuilding
- Whether the widget/browser-calling entry point (`handleWidgetCall`, `TWILIO_TWIML_APP_SID`, Twilio Voice SDK access tokens) is needed at all for an in-house tool, or purely phone-number-based calling is sufficient — simplifies `twilio.service.ts` meaningfully if dropped.
- The hardcoded 5-minute (`300s`) absolute ceiling on `maxDurationSeconds` (both in the Agent schema `max` and re-clamped in `initiateOutboundCall`) — confirm this limit is still wanted or should be raised for in-house use cases (e.g. long consultative calls).

## 13. Scale Requirement: 100,000+ calls/day, 1,000+ concurrent

This is a **hard target for the rebuild**, not a hypothetical — the platform must sustain roughly 100k+ calls/day with 1,000+ of them concurrent at peak (raised from an earlier 25k/day, 400+ concurrent target). This changes several "fine for a single instance" conclusions elsewhere in this doc, and at this scale a **multi-instance deployment should be assumed from the start**, not treated as a maybe-later fallback:

- **`sessions` Map (media-stream handler) and `callSessions` Map (Gather-mode)** — both in-process, per-instance. At 1,000+ concurrent calls, a single instance would need to hold 1,000+ live WebSocket connections (to both Twilio and Deepgram simultaneously, i.e. ~2,000 sockets) plus run the streaming-LLM/TTS pipeline for each — each session's own memory footprint is small (a few KB of history + a handful of open sockets), but **CPU/network throughput for concurrent TTS synthesis and audio streaming is the far more likely bottleneck than memory** at this concurrency. This state **must** move to Redis (or the deployment must shard calls across instances with sticky routing, e.g. consistent hashing on `callId` at the load balancer) — a real architectural change, not a config tweak, and effectively mandatory rather than optional at 1,000+ concurrent.
- **`cartesiaActiveCalls` concurrency counter** (`CARTESIA_CONCURRENT_LIMIT`, default 12) — this is a **hard ceiling on simultaneous Cartesia TTS calls**, tied to whatever the Cartesia plan actually allows. At 1,000+ concurrent calls all potentially wanting Cartesia voices, this limit will be hit constantly, meaning most calls fall back to OpenAI TTS. Either (a) budget for a much higher-tier Cartesia plan and raise this env var to match, or (b) accept that OpenAI TTS is the effective default at this scale and treat Cartesia as a limited "premium voice" pool rather than the primary path.
- **TTS pre-generated audio files on local disk** (`ttsAudio.service.ts`, `/tmp/tts-audio/`) — already flagged as instance-local; with 1,000+ concurrent calls necessarily spread across many instances, this **must** move to shared storage (S3, per the team's existing bucket) or be regenerated on-demand per-instance (the pattern the stream-mode greeting path already uses) rather than assuming the writing instance also serves the file.
- **Twilio account concurrency limits** — Twilio accounts have their own concurrent-call ceilings (both an overall account limit and per-number limits) that are entirely independent of this codebase; confirm the Twilio account backing the rebuild is provisioned/upgraded for 1,000+ concurrent calls before assuming the application layer is the only thing that needs scaling — this is very likely an enterprise-tier Twilio arrangement, not a default account limit.
- **Deepgram concurrent-connection limits** — same category of external-provider ceiling as Twilio/Cartesia; verify the Deepgram plan supports 1,000+ concurrent streaming connections.
- **MongoDB connection pool / write throughput** — every turn of every call writes a `transcriptEntries` push; at 1,000+ concurrent calls with multiple turns/minute each, this is a sustained, heavy write load. Size the Mongoose connection pool (likely well beyond the current `maxPoolSize:100` default, see `11-app-bootstrap-infra.md §8`) and (if self-hosting Mongo rather than Atlas) the underlying instance/cluster accordingly; this is a capacity-planning item, not a code change, but at this scale it warrants sharding/cluster sizing discussion, not just a pool-size bump.
- **Redis** — used for the KB embedding cache and the workflow-followup sorted set (see `06-workflows.md`, `07-knowledge-base.md`); at this scale, confirm the Redis instance's connection limit and throughput headroom, especially since `makeBullmqQueueConnection()` opens a fresh connection per enqueue (see `09-jobs-queues.md`) — 100k calls/day's worth of workflow-followup scheduling means a substantial number of short-lived Redis connections per day; monitor for connection-churn related throttling from the Redis provider and consider a pooled/shared connection strategy if churn becomes a bottleneck.

**Bottom line**: most of the call-pipeline logic itself (streaming LLM, sentence-boundary TTS, barge-in) doesn't need to change to hit this scale — the changes needed are almost entirely in **deployment topology and external-provider plan sizing**, not application code, with the one clear exception of TTS file storage (must become instance-independent) and session state (must move to Redis/shared storage — no longer optional at 1,000+ concurrent). Multi-instance-with-shared-state should be treated as the target architecture, not a stretch goal. See `11-app-bootstrap-infra.md` for how this ties into the overall deployment plan.
