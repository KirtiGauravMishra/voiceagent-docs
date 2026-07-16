# Backend Prompt 02 — Agents

> Read `../00-master-rebuild-guide.md` and `01-auth.md` first.

## Goal
Build the `Agent` resource — the configurable AI persona (LLM + voice + call behavior + post-call tasks) that calls, campaigns, and workflows all reference.

## Data model — `Agent`

Top-level: `name` (required, max 100), `description` (max 500), `language` (default `'en-US'`), `systemPrompt` (required, max 32000), `ownerId` (ObjectId → User, required), `status` (enum `active|inactive`, default `active`), `knowledgeDocIds` (ObjectId[] → KnowledgeDocument), `calcomEventTypeId` (number, optional — Cal.com integration), `calcomTimeZone` (string, default `'UTC'`).

**`llm`** (embedded): `provider` (`openai|anthropic`, default `openai`), `model` (free-text string, default `gpt-4.1-mini`), `temperature` (0–2, default 0.7), `maxTokens` (50–4096, default 400).

**`voice`** (embedded): `provider` (`openai|cartesia|twilio-polly|elevenlabs`, default `openai`), `voiceId` (string, default `alloy`), `speed` (0.5–2.0, default 1.0). No plan-gating on ElevenLabs — every provider is available to every agent (no billing system in this rebuild).

**`callSettings`** (embedded): `maxDurationSeconds` (30–300, default 300 — hard call-length ceiling, revisit if longer calls are needed for this org), `recordingEnabled` (bool), `transcriptionEnabled` (bool, default true), `greetingMessage` (max 500), `thinkingMessage` (max 200 — filler line spoken if the LLM takes >1.5s to respond), `hangupPrompt` (max 2000 — plain-English rules telling the LLM when to end the call), `farewellMessage` (max 300), `voicemailDetection` (bool), `retryOnNoAnswer` (bool), `maxRetries` (0–3, default 1), `phoneNumberId` (ObjectId → PhoneNumber — preferred caller ID), `transferEnabled` (bool), `transferPhoneNumber` (E.164, max 20), `transferMessage` (max 300), `transferConditions` (max 1000 — plain-English rules for when to transfer to a human).

**`taskSettings`** (embedded, post-call processing): `summarizationEnabled` (bool), `extractionEnabled` (bool), `extractionPrompt` (max 3000, seed with a default 4-field prompt: name/purpose/action/sentiment), `webhookUrl` (optional — POST extracted results here after each call), `successEvaluationEnabled` (bool).

Add indexes: `{ownerId, status, createdAt:-1}`, `{ownerId, _id}`. `toJSON` transform per master guide.

## Service-layer behavior

**Partial-update merge semantics (important)**: when handling `PATCH`, merge `llm`, `voice`, `callSettings`, and `taskSettings` **field-by-field against the current stored sub-document**, not a blanket overwrite. A `PATCH {callSettings:{farewellMessage:"..."}}` must not wipe every other `callSettings` field. Do this for every embedded sub-document.

**Caching (optional but recommended given scale)**: cache `getAgentById`/`getUserAgents` (page 1 only) in Redis with a short TTL (~1–5 min) and explicit invalidation on every create/update/delete — this is read-heavy, low-churn data, worth caching at this platform's request volume. Re-check ownership against the cached value too (don't let a cache hit leak another user's agent).

**No agent-count limit, no ElevenLabs gating** — there is no billing/plan system in this rebuild, so skip any "max agents per plan" or "premium voice" checks entirely.

## API — `/api/agents` (JWT required)

| Method | Path | Body | Purpose |
|---|---|---|---|
| GET | `/` | `?page=&limit=` (max 50) | paginated list, newest first |
| POST | `/` | full agent shape | create |
| GET | `/:id` | — | get one (403 if not owner) |
| PATCH | `/:id` | partial | merge-update (see above) |
| DELETE | `/:id` | — | delete |
| POST | `/:id/test-transfer` | `{toNumber?}` | places a **real** live bridge call to verify human-handoff wiring — dial `toNumber` (or the agent's configured `transferPhoneNumber` if omitted), speak a short test message, then `<Dial>` to the transfer number, exactly like a production handoff |
| POST | `/:id/widget-icon` | multipart `file` (≤5MB) | upload an icon image to S3, store the returned URL — only build this if you're keeping any agent-branding feature; skip if not needed |
| `*` | `/:agentId/calls/*` | — | mount the Calls router (Prompt 03) scoped to this agent |

Validate `name` (2–100 chars) and `systemPrompt` (10–32000 chars) as required on create; everything else optional with the defaults above, mirrored in both the Mongoose schema and the request-validation schema.

## Environment variables
None specific to this module beyond the global LLM/voice provider keys (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `CARTESIA_API_KEY`, `ELEVENLABS_API_KEY`) and AWS Polly credentials (voice preview only) from the master guide.

## Acceptance checklist
- [ ] Creating an agent with only `name` + `systemPrompt` succeeds with every other field defaulted correctly.
- [ ] `PATCH` with a partial `callSettings` object only changes the given fields, leaving siblings intact.
- [ ] `test-transfer` places a real Twilio call and bridges to the configured number.
- [ ] Ownership is enforced on every read/write (a second user's token gets 403, not the agent).
