# 07 — Knowledge Base (RAG) & Knowledge Gaps

> Backend domain doc. Source: `backend/src/{models/{KnowledgeDocument,KBChunk,KnowledgeGap}.model.ts, controllers/{knowledge,knowledgeGap}.controller.ts, routes/{knowledge,knowledgeGap}.routes.ts, services/{knowledge,knowledgeGap}.service.ts}`.

## 1. Overview

Two related but independent features:
- **Knowledge Base** — customers upload documents (PDF/TXT/MD); the system chunks and embeds them, and agents can semantically search that content mid-call/mid-chat to ground their answers (classic RAG).
- **Knowledge Gaps** — an analytics feature that mines call transcripts for moments the agent had to fall back to "I don't know" / "I'm not sure", surfacing them as a prioritized worklist so the customer knows what to add to their knowledge base next.

## 2. Data Models

### `KnowledgeDocument` (`models/KnowledgeDocument.model.ts`) — one row per uploaded file
| Field | Type | Notes |
|---|---|---|
| `ownerId` | ObjectId → User | indexed |
| `ragId` | string (unique) | a stable public UUID (separate from Mongo `_id`) shown in the UI/logs — decouples the externally-visible ID from the internal document ID |
| `filename` | string | |
| `status` | enum | `processing \| processed \| failed` |
| `ext` | enum | `pdf \| txt \| md \| other` |
| `fileSize` | number | bytes |
| `chunkCount` | number | set once processing finishes |
| `errorMsg` | string | populated only on `failed` |

### `KBChunk` (`models/KBChunk.model.ts`) — the actual RAG content, in a **separate collection** specifically to avoid MongoDB's 16MB per-document limit (a document's full chunk set could otherwise blow past that embedded in the parent doc)
| Field | Type | Notes |
|---|---|---|
| `docId` | ObjectId → KnowledgeDocument | |
| `ownerId` | ObjectId → User | denormalized for owner-scoped cleanup without a join |
| `text` | string | the chunk's raw text, injected into the LLM prompt verbatim when retrieved |
| `embedding` | number[] | 1536-dim OpenAI `text-embedding-3-small` vector |
| `chunkIndex` | number | position within the source document |

No `timestamps`. Indexes: `{docId:1,chunkIndex:1}` (ordered chunk retrieval), `{ownerId:1}` (bulk owner cleanup).

### `KnowledgeGap` (`models/KnowledgeGap.model.ts`)
| Field | Type | Notes |
|---|---|---|
| `ownerId` / `agentId` | ObjectId | |
| `question` | string | the actual user utterance that triggered a fallback (display copy — kept as the *longer* of any two duplicate variants seen) |
| `normalizedQuestion` | string | lowercased, punctuation-stripped, whitespace-collapsed — the **dedup key** |
| `status` | enum | `pending \| resolved \| ignored` |
| `severity` | number | 1–10, see §4 |
| `count` | number | how many distinct calls have hit this same gap |
| `firstSeenAt` / `lastSeenAt` | Date | |
| `lastCallId` / `seenCallIds[]` | ObjectId | `seenCallIds` capped at the most recent 200 to bound document size |

Unique compound index `{ownerId,agentId,normalizedQuestion}` — this **is** the upsert key (see §4), so the same question asked in different calls accumulates onto one row rather than creating duplicates.

## 3. Knowledge Base — upload & processing pipeline (`knowledge.service.uploadDocument`)

### API — `/api/knowledge` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | List the owner's documents |
| POST | `/upload` | Upload (multer, 10MB limit, mimetype/extension-filtered to pdf/txt/md) |
| GET | `/:id/status` | Poll processing status (frontend polls this after upload since processing is async) |
| DELETE | `/:id` | Delete document + all its chunks |

### Pipeline (fully async — the HTTP response returns immediately after creating the `KnowledgeDocument` row with `status:'processing'`; the actual work runs in a `setImmediate` background callback)
1. **Extract text**: PDF → `pdf-parse` (dynamically `require`'d, wrapped in try/catch — a parse failure yields empty text rather than crashing); TXT/MD → raw UTF-8 buffer decode.
2. **Chunk** (`chunkText`, `CHUNK_CHARS=1800`, `OVERLAP_CHARS=200`): paragraph-aware — splits on blank lines, accumulates paragraphs into ~1800-char chunks, and **carries the last 200 chars of each chunk forward as the start of the next** (a manual sliding-window overlap so semantic context isn't lost at chunk boundaries). Any single paragraph longer than the chunk size is further split on sentence boundaries (`.`/`!`/`?`/`…`).
3. **Embed** (`embedTexts`): batches up to 100 chunks per OpenAI `text-embedding-3-small` call, with a 300ms pause between batches (rate-limit hygiene) and exponential backoff (up to 4 retries, 1s→16s) specifically on HTTP 429/503. A batch that fails all retries stores its chunks with an **empty embedding array** rather than aborting the whole upload — those chunks silently fall back to keyword-substring matching at search time instead of vector similarity (see §5).
4. **Query-embedding cache**: single-text embed calls (i.e. search queries, not document chunks) are cached in Redis for 2 hours keyed by `sha256(text)` — embeddings are deterministic, so this is a pure latency/cost optimization with no staleness risk.
5. **Bulk insert** chunks into `KBChunk` in batches of 500 (`insertMany(..., {ordered:false})`).
6. **Finalize**: flips `KnowledgeDocument.status → 'processed'`, sets `chunkCount`. On any failure at any stage: `status → 'failed'` + `errorMsg`, and any partially-inserted chunks are deleted (no orphaned half-processed chunks left behind).

## 4. Semantic search at call/chat time (`knowledge.service.searchKnowledge`)

`searchKnowledge(query, docIds, topK=6)` — called from three places: the Gather-mode Twilio pipeline, the real-time media-stream pipeline (both `03-calls-realtime-pipeline.md`), and the widget chat reply generator.

1. Caps `docIds` at 20 (`MAX_DOCS_PER_SEARCH`) — a safety limit on how many documents one query can span.
2. Embeds the query (cache-checked first, per above).
3. **Attempt 1 — MongoDB Atlas `$vectorSearch`**: an aggregation pipeline stage doing index-backed approximate-nearest-neighbor search, filtered to the given `docId`s, `numCandidates = max(topK*15, 150)`. Requires a pre-created Atlas Search index (`ensureAtlasVectorIndex()`, called once at API startup from `index.ts` — idempotent, logs and continues if the index already exists or if Atlas doesn't support it at all).
4. **Attempt 2 — JS cosine-similarity fallback**: if Atlas search isn't available (self-hosted/local MongoDB, or a free-tier Atlas cluster without Search) or returns nothing, loads **every** chunk for the given docs into memory and computes cosine similarity by hand (`cosineSim`) — an `O(n)` in-process scan, fine at small-to-moderate document-set sizes but does not scale the way Atlas's ANN index does. If the query embedding itself is empty (OpenAI unavailable), falls back further to a plain case-insensitive substring match as the "score."
5. Results below a `score > 0.15` threshold are discarded; surviving chunks are formatted as `"[Context N]\n{text}"` blocks joined by `---` separators and returned as one string, ready to be appended to the LLM system prompt.

The module caches whether Atlas vector search is available (`_atlasVectorAvailable`, tri-state null/true/false) after the first successful/failed attempt, so it doesn't retry the expensive-to-fail Atlas path on every single search once it's known to be unsupported in the current environment.

## 5. Knowledge Gap detection (`knowledgeGap.service.detectAndUpsertKnowledgeGapsForCall`)

Called from `postCall.service.processPostCall` after every completed call (gated only by `UserSettings.knowledgeGapAnalysisEnabled`, default true — independent of the agent's `taskSettings` toggles).

1. **Signal extraction** (`extractSignalsFromCall`): walks `Call.transcriptEntries` turn by turn. Tracks the last thing the *user* said (≥7 chars) as `lastUserQuestion`. Whenever the **agent's** reply matches one of several hardcoded fallback-phrase regexes (`"I don't know"`, `"I'm not sure"`, `"I can't help"`, `"that information isn't available"`, `"please contact support"`, `"could you repeat that"`, etc.), the preceding user question is captured as a gap signal, normalized, and deduped **within that single call** (a `Map` keyed by normalized text — the same question asked twice in one call only produces one signal).
2. **Severity scoring**: a `computeBaseSeverity` (4 baseline; +2 if the call's overall `status` was `failed/no-answer/busy/voicemail`; +2 if `callScore < 40`, or +1 if `< 70`) combined with `computeSeverity(base, totalCount)` which adds `floor(totalCount/2)` — i.e. **severity rises the more times a gap has been hit across all calls**, capped at 10.
3. **Upsert** keyed on the unique `{ownerId, agentId, normalizedQuestion}` index: a brand-new gap is created at `count:1`; a recurring one increments `count`, extends `seenCallIds` (capped to the most recent 200), bumps `lastSeenAt`, takes the **longer** of the old/new question text for display, recomputes severity (keeping whichever is higher — severity never decreases from a fresh detection), and — notably — **resets `status` back to `'pending'`** and clears `resolvedAt`/`ignoredAt` even if a human had previously marked it `resolved`/`ignored`. This is a deliberate behavior: if a "resolved" gap gets hit again by a real caller, it's surfaced again rather than staying silently closed.
4. A given call is only counted once per gap (`seenCallIds` membership check) — reprocessing the same call (e.g. via the recompute batch job) doesn't inflate `count`.

### Manual recompute (`recomputeKnowledgeGaps`, `POST /api/knowledge-gaps/recompute`)
Re-runs gap detection over the owner's last N days of calls (clamped 1–180, default presumably passed from the frontend), capped at the most recent 2000 calls with any transcript. Useful after enabling the feature retroactively, or after tuning fallback-phrase detection.

## 6. API Reference — `/api/knowledge-gaps` (JWT)

| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Paginated list — filters `status`, `agentId`, `dateFrom`/`dateTo`; `sortBy: recent\|severity\|oldest` |
| GET | `/stats` | `{total, pending, resolved, ignored, highSeverity (severity≥8), avgSeverity, agents:[{id,name,count}]}` — dashboard summary |
| POST | `/recompute` | Re-scan recent calls (see above) |
| PATCH | `/:id/status` | Mark `pending\|resolved\|ignored` — stamps `resolvedAt`/`ignoredAt` accordingly, clears the other |

## 7. Rebuild Notes (Replit target)

### No billing coupling
Neither Knowledge Base nor Knowledge Gaps have any dependency on the billing/subscription system — pure feature logic, nothing to strip for an in-house rebuild.

### Dependencies
`pdf-parse` (PDF text extraction — note the exact pinned version `1.1.1` in `package.json`, not the `^` range used elsewhere, likely pinned due to a breaking change in later versions), `openai` SDK (embeddings), `mongoose`, `ioredis` (query-embedding cache only — non-fatal if Redis is briefly unavailable, per `getCachedEmbedding`'s try/catch-returns-null).

### Infrastructure decision: MongoDB Atlas vs. self-hosted/other Mongo
The vector search quality/performance depends entirely on whether the database is **MongoDB Atlas** (with Atlas Search enabled) vs. self-hosted MongoDB or a different provider:
- **On Atlas**: `ensureAtlasVectorIndex()` (called once at boot) provisions a `$vectorSearch`-compatible index automatically; searches are ANN-indexed and fast even at scale.
- **Off Atlas** (e.g. a plain `mongod` container on Replit, or MongoDB via a non-Atlas host): the JS cosine-similarity fallback activates automatically and transparently — **no code changes needed** — but it's an `O(n)` full-scan per search across every chunk in the referenced documents, which will degrade as knowledge bases grow large. If the in-house rebuild won't use Atlas, decide up front whether this fallback's scaling ceiling is acceptable, or whether a dedicated vector DB (e.g. pgvector, Qdrant, self-hosted) should replace it — that would be a non-trivial rewrite of `atlasVectorSearch`/`searchKnowledge`, not a config change.

### Env vars relevant to this domain
`OPENAI_API_KEY` (required — both extraction embeddings and query embeddings; without it, `embedTexts` returns empty vectors for everything and search silently degrades to keyword substring matching only), `REDIS_URL` (query-embedding cache; optional — degrades gracefully to "always call OpenAI" if absent), `MONGODB_URI` (must point at Atlas if you want native vector search).

### Storage note
Uploaded file **bytes themselves are never persisted** — only the extracted/chunked/embedded text lives in `KBChunk`. The original PDF/TXT/MD buffer exists only transiently in memory during the multer upload + `setImmediate` processing step. This means there's nothing to migrate to S3 for this feature specifically (unlike the Cloudinary-based agent widget-icon upload in `02-agents.md`) — no rebuild action needed here.
