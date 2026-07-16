# 02 — Agents

> Backend domain doc. Source: `backend/src/{models/Agent.model.ts, controllers/agent.controller.ts, services/agent.service.ts, routes/agent.routes.ts, validations/agent.validation.ts}`.

## 1. Overview

An **Agent** is the configurable unit that defines one AI voice/chat persona: its system prompt, which LLM and voice provider power it, how it behaves on a call (greeting, hangup rules, transfer-to-human rules, recording), what it does after a call (summarize/extract/score), and its embeddable widget appearance. Every `Call`, `Campaign`, and `Workflow` "call" node references an `Agent`. Agents are strictly owner-scoped — every read/write is filtered by `ownerId === req.user.id`, and cross-tenant access attempts return `403 Access denied` (not `404`, so ownership-mismatch is distinguishable from not-found in logs, though both should probably look identical to the end user).

## 2. Data Model (`models/Agent.model.ts`)

Top-level fields:

| Field | Type | Notes |
|---|---|---|
| `name` | string | required, max 100 |
| `description` | string | optional, max 500 |
| `language` | string | default `'en-US'` |
| `systemPrompt` | string | required, max 32000 chars — the core LLM instruction set |
| `ownerId` | ObjectId → User | required |
| `status` | enum | `'active' \| 'inactive'` — inactive agents are rejected at call-dispatch time |
| `knowledgeDocIds` | ObjectId[] → KnowledgeDocument | which RAG documents this agent can search (see `07-knowledge-base.md`) |
| `calcomEventTypeId` | number | optional — Cal.com event type ID, enables AI booking tool-calls when the owner has a Cal.com connection |
| `calcomTimeZone` | string | default `'UTC'` — IANA tz used for Cal.com slot queries |

### `llm` (embedded, `_id:false`)
| Field | Default | Notes |
|---|---|---|
| `provider` | `'openai'` | `'openai' \| 'anthropic'` |
| `model` | `'gpt-4.1-mini'` | free-text — not validated against a provider's actual model list |
| `temperature` | `0.7` | 0–2 |
| `maxTokens` | `400` | 50–4096 |

### `voice` (embedded)
| Field | Default | Notes |
|---|---|---|
| `provider` | `'openai'` | `'openai' \| 'cartesia' \| 'twilio-polly' \| 'elevenlabs'` — see cost comments in the model file: OpenAI ~$0.015/min, Cartesia ~$0.02/min, Polly free-with-Twilio |
| `voiceId` | `'alloy'` | provider-specific: OpenAI voice name, `Polly.Joanna`-style for twilio-polly, a Cartesia voice UUID, or an ElevenLabs voice ID |
| `speed` | `1.0` | 0.5–2.0 |
| `stability` / `similarityBoost` | — | reserved, not currently wired to any TTS call (ElevenLabs supports these but the code doesn't pass them through yet — confirm before relying on them in a rebuild) |

**`elevenlabs` is plan-gated** — `agent.service.createAgent`/`updateAgent` calls `planAllowsElevenLabs(userId)` (in `billing.service.ts`) and throws `403 ELEVENLABS_NOT_ALLOWED` if the current plan doesn't include it. **For an in-house rebuild with no billing/plan system, this gate should simply be deleted** (or hardcoded to always-allow) — see §6.

### `callSettings` (embedded)
| Field | Default | Notes |
|---|---|---|
| `maxDurationSeconds` | `300` | 30–300 (hard 5-minute ceiling baked into the schema `max`) — this is also the fallback used to estimate duration for stuck-call cleanup (see `03-calls-realtime-pipeline.md`) |
| `recordingEnabled` | `false` | |
| `transcriptionEnabled` | `true` | |
| `greetingMessage` | — | max 500 — first line the agent speaks |
| `thinkingMessage` | — | max 200 — filler line played if the LLM hasn't produced a first sentence within 1.5s (streaming pipeline only) |
| `hangupPrompt` | — | max 2000 — plain-English instructions telling the LLM *when* to end the call; the LLM signals this by emitting a `[HANGUP]` sentinel (see call-pipeline doc) |
| `farewellMessage` | — | max 300 — spoken just before hangup |
| `voicemailDetection` | `false` | passed to Twilio's AMD (`machineDetection`) on outbound dispatch |
| `retryOnNoAnswer` / `maxRetries` | `false` / `1` (0–3) | used by the campaign dialer, not single ad-hoc calls |
| `phoneNumberId` | — | ref → PhoneNumber — the agent's preferred outbound caller ID |
| `transferEnabled` | `false` | allows the LLM to trigger a live transfer to a human via `[TRANSFER]` sentinel |
| `transferPhoneNumber` | — | max 20, E.164 |
| `transferMessage` | — | max 300 — spoken right before bridging |
| `transferConditions` | — | max 1000 — plain-English rules fed into the system prompt telling the LLM *when* to transfer |

### `taskSettings` (embedded, post-call processing — see `postCall.service.ts`, referenced in call-pipeline doc)
| Field | Default | Notes |
|---|---|---|
| `summarizationEnabled` | `false` | LLM summarizes the transcript after the call |
| `extractionEnabled` | `false` | LLM extracts structured fields per `extractionPrompt` |
| `extractionPrompt` | a default 4-field prompt (name/purpose/action/sentiment) | max 3000, fully overridable |
| `webhookUrl` | — | optional — POST extracted data here after each call completes |
| `successEvaluationEnabled` | `false` | LLM scores 0–100 whether the call achieved its goal |

### `voiceWidget` (embedded — for the browser-embeddable chat/voice widget)
| Field | Default | Notes |
|---|---|---|
| `isEnabled` | `false` | |
| `iconUrl` | — | uploaded via a dedicated endpoint (see §4) |
| `primaryColor` | `'#8B5CF6'` | |
| `position` | `'bottom-right'` | or `'bottom-left'` |
| `callRestrictions.enabled` | `false` | abuse-prevention for the *public* voice widget |
| `callRestrictions.maxCallsPerIp` | `3` | 1–100, within `windowHours` |
| `callRestrictions.windowHours` | `24` | 1–168 |
| `callRestrictions.requireCaptcha` | `false` | shows a math challenge before allowing a voice call |

> Since the public embeddable widget is **out of scope for the in-house rebuild** (per the earlier scoping decision), `voiceWidget`/`callRestrictions` can likely be dropped entirely from a rebuilt schema unless an internal-only chat/voice widget is still wanted. Kept here for completeness since the field still lives on the same `Agent` document today.

### Indexes
```js
{ ownerId: 1, status: 1, createdAt: -1 }
{ ownerId: 1, _id: 1 }
{ ownerId: 1, createdAt: -1 }  // partial: status:'active' only
```
All three support the common "list my active agents, newest first" query pattern without a collection scan.

### `toJSON` transform
Standard `_id → id`, strips `__v`. No fields are hidden (unlike `User`, nothing here is a secret).

## 3. Service Layer (`services/agent.service.ts`)

### Caching
Agent reads are cached in an **in-process, per-instance** LRU (`utils/cache.ts` — see rebuild note below), not Redis:
- `getUserAgents(userId, page, limit, skip)` — only caches the very common **page 1 / limit 10** shape (`CacheKeys.userAgents(userId)`, TTL 300s ± 10% jitter). Any other page/limit combination bypasses the cache and hits Mongo directly.
- `getAgentById(agentId, userId)` — cached per-agent (`CacheKeys.agent(agentId)`, TTL 120s ± jitter), with the ownership check re-applied against the **cached** value too (so a cache hit can't leak another user's agent).
- Every mutation (`createAgent`, `updateAgent`, `deleteAgent`) calls `cacheDel(...)` on the relevant keys — cache invalidation is explicit, not TTL-only.
- The cache module's own comment explains the rationale: with 2 horizontally-scaled instances (Render, historically), in-memory reads (0ms) beat a remote Redis round-trip (200–300ms) for this read-heavy, small-blast-radius data; worst case a stale entry lives for the TTL. **This reasoning doesn't change for a single-instance Replit deployment** — in-memory is still simpler and faster; only revisit if the rebuild ever runs multiple concurrent instances of the API process (in which case two instances could serve stale vs. fresh agent data for up to the TTL window after an edit).

### Partial-update merge semantics (`updateAgent`)
PATCH requests don't need to send whole sub-documents — `updateAgent` manually merges `llm`, `voice`, `callSettings`, and `voiceWidget` (including its nested `callRestrictions`) field-by-field against the **current stored values** before assigning, specifically so that e.g. `PATCH { callSettings: { farewellMessage: "..." } }` doesn't blow away every other `callSettings` field. This hand-rolled merge (rather than relying on Mongoose's default `Object.assign` behavior for nested schemas, which would replace the whole sub-object) is a deliberate pattern — **replicate it in a rebuild** or partial agent edits will silently wipe sibling settings.

### Plan gating (billing-dependent — irrelevant for in-house rebuild)
`createAgent` checks `Subscription` → `Plan.maxAgents` and throws `403 AGENT_LIMIT_REACHED` if the owner is at their plan's agent cap. Both `createAgent` and `updateAgent` gate `voice.provider === 'elevenlabs'` behind `planAllowsElevenLabs()`. **Since billing/subscriptions are excluded from this rebuild, both gates should be removed** (or replaced with a flat internal policy, e.g. "no cap" / "ElevenLabs always allowed") — see §6.

## 4. API Reference — `/api/agents` (JWT required on every route)

| Method | Path | Body / Query | Purpose |
|---|---|---|---|
| GET | `/` | `?page=&limit=` (limit capped at 50, default 10) | Paginated list of the caller's agents, newest first |
| POST | `/` | `createAgentSchema` | Create an agent |
| GET | `/:id` | — | Get one agent (403 if not yours) |
| PATCH | `/:id` | `updateAgentSchema` (all fields optional) | Partial update, see merge semantics above |
| DELETE | `/:id` | — | Delete |
| POST | `/:id/test-transfer` | `{ toNumber? }` | Places a **real** live bridge call to `toNumber` (or the agent's configured `transferPhoneNumber` if omitted) to verify human-handoff wiring end-to-end before going live. Delegates to `twilio.service.testCallTransfer` — see call-pipeline doc. |
| POST | `/:id/widget-icon` | multipart `file` (≤5MB) | Uploads to Cloudinary (`cloudinary.service.uploadImageBuffer`), falling back to an inline base64 `data:` URL if Cloudinary isn't configured or fails; sets `voiceWidget.iconUrl` via the normal `updateAgent` path |
| `*` | `/:agentId/calls/*` | — | Mounts `call.routes` with `agentId` bound from the parent route param — i.e. every call-domain route is also reachable scoped to one agent (e.g. `GET /api/agents/:agentId/calls`) |

There is also an **unauthenticated** `GET /api/agents/:id/widget-config`-style endpoint implemented in the controller (`getWidgetConfig`) that returns only `{isEnabled, iconUrl, primaryColor, position, agentName}` for public widget rendering — confirm its exact mount path in `agent.routes.ts` before relying on it; as of this read it's exported from the controller but the route file shown does not explicitly wire it under `/api/agents`, so it may be mounted elsewhere (likely alongside the widget router) or currently dead code — **flag for verification during the rebuild** rather than assuming it's live.

### Validation (Yup, `validations/agent.validation.ts`)
`createAgentSchema` requires `name` (2–100 chars) and `systemPrompt` (10–32000 chars); everything else (llm/voice/callSettings/taskSettings/voiceWidget/calcom fields) is optional with schema-level defaults mirroring the Mongoose schema defaults. `updateAgentSchema` is the same shape with `name`/`systemPrompt` also made optional (PATCH semantics). Nested objects are validated as whole sub-schemas but individual sub-fields are optional, matching the merge-not-replace behavior in the service.

## 5. How Agent config feeds the runtime call pipeline

(Full detail in `03-calls-realtime-pipeline.md` — summarized here for context.) At call time, `Agent` is loaded once and its fields drive: which LLM/model/temperature to call, which TTS provider/voice to synthesize with, what greeting/thinking/farewell lines to speak, whether `[HANGUP]`/`[TRANSFER]` sentinels are enabled in the system prompt (only if `hangupPrompt`/`transferEnabled` are set), the max call duration (hard-disconnects at `maxDurationSeconds`), and which `knowledgeDocIds` are eligible for RAG search mid-call.

## 6. Rebuild Notes (Replit target, in-house/no-billing)

### Remove these billing-coupled checks
In `agent.service.ts`:
- `createAgent`'s `Subscription`/`Plan.maxAgents` lookup and the `AGENT_LIMIT_REACHED` throw (lines ~80–92) — delete entirely for an in-house build with no per-plan agent caps.
- The `planAllowsElevenLabs(userId)` gate in both `createAgent` and `updateAgent` — delete, or replace the import with a constant `true` if you want to keep the code path shape but always allow ElevenLabs.
- Once both are removed, the `Subscription`/`Plan` model imports at the top of `agent.service.ts` become unused and should be deleted too.

### Dependencies
`mongoose`, `yup`, `multer` (widget-icon upload only — irrelevant if the public widget is dropped), `cloudinary` SDK (icon upload — see storage note below).

### Storage swap (Cloudinary → S3)
`uploadWidgetIcon` currently uploads to Cloudinary. If the public widget feature is dropped for the in-house build (per the earlier scoping decision), this endpoint and its Cloudinary dependency can likely be deleted outright. If an icon-upload feature is still wanted internally, swap `cloudinary.service.uploadImageBuffer` for an S3 `PutObjectCommand` against the team's existing bucket — the call site in `agent.controller.ts::uploadWidgetIcon` only needs the returned public URL, so the swap is contained to one service file.

### Env vars relevant to this domain
None specific to Agents beyond the global `OPENAI_API_KEY` / `ANTHROPIC_API_KEY` / `CARTESIA_API_KEY` / `ELEVENLABS_API_KEY` / AWS Polly credentials that determine which `voice.provider`/`llm.provider` choices actually work at runtime (documented fully in `11-app-bootstrap-infra.md`).

### Things to decide before rebuilding
- Whether to keep the `voiceWidget`/public-widget fields on the schema at all, given the public widget API is out of scope.
- Whether an agent cap or ElevenLabs restriction is wanted for any *other* reason (e.g. cost control) even without a billing system — if so, replace the plan-based check with a simple env-var-driven constant rather than removing the concept entirely.
