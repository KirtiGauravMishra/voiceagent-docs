# Backend Prompt 07 — Knowledge Base (RAG) & Knowledge Gaps

> Read `../00-master-rebuild-guide.md`, `02-agents.md`, `03-calls-realtime-pipeline.md` first.

## Goal
Two related but independent features:
- **Knowledge Base** — upload documents (PDF/TXT/MD), chunk + embed them, and let an agent semantically search that content mid-call to ground its answers (RAG).
- **Knowledge Gaps** — mine call transcripts for moments the agent fell back to "I don't know," surfacing them as a prioritized worklist of what to add to the knowledge base next.

## Data model — `KnowledgeDocument` (one row per uploaded file)

`ownerId`, `ragId` (a stable public UUID, separate from the Mongo `_id` — this is the id shown in the UI/logs, decoupling the externally-visible id from the internal document id), `filename`, `status` (enum `processing|processed|failed`), `ext` (enum `pdf|txt|md|other`), `fileSize` (bytes), `chunkCount`, `errorMsg` (populated only on `failed`).

**Storage note**: don't persist the uploaded file's raw bytes anywhere — only the extracted/chunked/embedded text needs to live in the database. The original file buffer only needs to exist transiently in memory during upload + processing. Nothing to put in S3 for this feature.

## Data model — `KBChunk` (the actual RAG content — a separate collection, not embedded in `KnowledgeDocument`, specifically to avoid MongoDB's 16MB per-document limit)

`docId` (→KnowledgeDocument), `ownerId` (denormalized, for owner-scoped cleanup without a join), `text` (the chunk's raw text, injected into the LLM prompt verbatim when retrieved), `embedding` (number[], 1536-dim for OpenAI `text-embedding-3-small`), `chunkIndex` (position within the source document).

Indexes: `{docId:1, chunkIndex:1}` (ordered retrieval), `{ownerId:1}` (bulk owner cleanup). If using MongoDB Atlas, also create a `$vectorSearch`-compatible index on `embedding` (cosine similarity, 1536 dimensions) — provision it once at boot (idempotent — log and continue if it already exists or if the current Mongo isn't Atlas at all).

## Ingestion pipeline (`POST /api/knowledge/upload`)

Respond immediately after creating the `KnowledgeDocument` row with `status:'processing'` — run the actual work in the background (a `setImmediate` callback or a BullMQ job, per Prompt 09), never block the upload request on it.

1. **Extract text**: PDF → a PDF-text-extraction library, wrapped in try/catch (a parse failure should yield empty text, not crash the pipeline); TXT/MD → raw UTF-8 decode.
2. **Chunk** (paragraph-aware): split on blank lines, accumulate paragraphs into ~1800-character chunks, and carry the last ~200 characters of each chunk forward as the start of the next (sliding-window overlap so semantic context isn't lost at chunk boundaries). Split any single paragraph longer than the chunk size on sentence boundaries.
3. **Embed**: batch up to 100 chunks per embedding-API call, with a short pause between batches and exponential backoff (a few retries) specifically on 429/503 responses. If a batch exhausts its retries, store those chunks with an empty embedding array rather than aborting the whole upload — they'll fall back to keyword-substring matching at search time instead of vector similarity.
4. **Query-embedding cache**: cache single-text embed calls (i.e. search queries, not document chunks) in Redis for ~2 hours keyed by a hash of the text — embeddings are deterministic, so this is a pure latency/cost win with no staleness risk.
5. **Bulk insert** chunks into `KBChunk` in batches (`insertMany(..., {ordered:false})`).
6. **Finalize**: `status → 'processed'`, `chunkCount` set. On failure at any stage: `status → 'failed'` + `errorMsg`, and delete any partially-inserted chunks for that document — never leave a half-processed document queryable.

## Semantic search at call time (`searchKnowledge(query, docIds, topK=6)`)

Called mid-turn from the real-time media-stream pipeline (Prompt 03).

1. Cap `docIds` at a sane per-query limit (e.g. 20).
2. Embed the query (cache-checked first).
3. **Attempt 1 — MongoDB Atlas `$vectorSearch`**: an aggregation stage doing index-backed ANN search filtered to the given doc ids, `numCandidates = max(topK*15, 150)`.
4. **Attempt 2 — JS cosine-similarity fallback**: if Atlas search isn't available (self-hosted Mongo, or a tier without Search) or returns nothing, load every chunk for the given docs into memory and compute cosine similarity by hand. This is fine at small-to-moderate document-set sizes but doesn't scale like an ANN index — if a single agent's knowledge base grows into the tens of thousands of chunks, standing up Atlas Search (or another vector-index-capable store) is the right fix, not tuning the JS path further. If the query embedding itself is empty (embedding provider unavailable), fall back further to a plain case-insensitive substring match.
5. Discard results below a similarity-score threshold (e.g. 0.15) so irrelevant chunks aren't injected when nothing in the knowledge base is actually relevant.
6. Format surviving chunks as `"[Context N]\n{text}"` blocks joined by separators, ready to append to the LLM system prompt.

Cache whether Atlas vector search is available (tri-state: unknown/true/false) after the first attempt, so you don't retry an expensive-to-fail Atlas path on every single search once you know it's unsupported in the current environment.

**Latency budget**: this runs inside the live per-turn pipeline — keep `numCandidates`/`topK` small, reuse the embedding client connection, and apply the same pipeline-staleness check from Prompt 03 (discard the result if a newer turn has already started).

## API — `/api/knowledge` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | list the owner's documents |
| POST | `/upload` | upload (multer, ~10MB limit, mimetype/extension-filtered to pdf/txt/md) |
| GET | `/:id/status` | poll processing status (frontend polls this after upload since processing is async) |
| DELETE | `/:id` | delete document + all its chunks |

## Data model — `KnowledgeGap`

`ownerId`, `agentId`, `question` (the actual user utterance that triggered a fallback — display copy, kept as the longer of any two duplicate variants seen), `normalizedQuestion` (lowercased, punctuation-stripped, whitespace-collapsed — the dedup key), `status` (enum `pending|resolved|ignored`), `severity` (1–10), `count` (how many distinct calls have hit this same gap), `firstSeenAt`/`lastSeenAt`, `lastCallId`, `seenCallIds[]` (capped at the most recent 200, to bound document size).

Unique compound index `{ownerId, agentId, normalizedQuestion}` — this is the upsert key, so the same question asked across different calls accumulates onto one row instead of creating duplicates.

## Knowledge gap detection (run after every completed call, from post-call processing — Prompt 03 §10)

Gate this behind a per-user setting (default enabled), independent of any per-agent task-setting toggles.

1. **Signal extraction**: walk the call's transcript turn by turn. Track the last thing the user said (≥7 chars) as the candidate question. Whenever the agent's reply matches one of several fallback-phrase patterns ("I don't know", "I'm not sure", "I can't help", "that information isn't available", "please contact support", "could you repeat that", etc.), capture the preceding user question as a gap signal, normalize it, and dedupe **within that single call** (so the same question asked twice in one call only produces one signal).
2. **Severity scoring**: a baseline score (e.g. 4), +2 if the call's overall outcome was unsuccessful (`failed`/`no-answer`/`busy`/`voicemail`), +2 if the call's quality score is very low (or +1 if moderately low), combined with a bump based on total historical hit count (e.g. `floor(totalCount/2)`) — i.e. severity rises the more times a gap has been hit across all calls, capped at 10.
3. **Upsert** keyed on `{ownerId, agentId, normalizedQuestion}`: a brand-new gap is created at `count:1`; a recurring one increments `count`, extends `seenCallIds` (capped at 200), bumps `lastSeenAt`, keeps the longer of the old/new question text for display, recomputes severity keeping whichever is higher (severity never decreases from a fresh detection), and **resets `status` back to `pending`** (clearing `resolvedAt`/`ignoredAt`) even if a human had previously marked it resolved/ignored — a gap that resurfaces in a real call should come back onto the worklist rather than staying silently closed.
4. A given call only counts once per gap (check `seenCallIds` membership) — reprocessing the same call (e.g. via manual recompute) must not inflate `count`.

### Manual recompute (`POST /api/knowledge-gaps/recompute`)
Re-run gap detection over the owner's last N days of calls (clamp 1–180), capped at a bounded number of most-recent calls with a transcript (e.g. 2000) so a single recompute call can't run unbounded. Useful after enabling the feature retroactively or after tuning fallback-phrase detection.

## API — `/api/knowledge-gaps` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | paginated list — filters `status`, `agentId`, `dateFrom`/`dateTo`; `sortBy: recent\|severity\|oldest` |
| GET | `/stats` | `{total, pending, resolved, ignored, highSeverity (severity≥8), avgSeverity, agents:[{id,name,count}]}` |
| POST | `/recompute` | re-scan recent calls (see above) |
| PATCH | `/:id/status` | mark `pending\|resolved\|ignored` — stamp `resolvedAt`/`ignoredAt` accordingly, clear the other |

## Infrastructure decision: MongoDB Atlas vs. self-hosted

Vector search quality/performance depends entirely on whether the database is MongoDB Atlas with Atlas Search enabled:
- **On Atlas**: provision a `$vectorSearch`-compatible index once at boot; searches are ANN-indexed and fast even at scale.
- **Off Atlas**: the JS cosine-similarity fallback activates automatically — no code changes needed — but it's an `O(n)` full scan per search across every chunk in the referenced documents, which degrades as knowledge bases grow. If this rebuild won't run on Atlas, decide up front whether that scaling ceiling is acceptable at your target volume, or whether a dedicated vector store (pgvector, Qdrant, etc.) should replace it — a real rewrite of the search function, not a config change.

## Environment variables
`OPENAI_API_KEY` (required — embeddings for both ingestion and query-time search; without it, embedding calls return empty vectors and search silently degrades to keyword substring matching only), `REDIS_URL` (query-embedding cache — optional, degrades gracefully to "always call the embedding API" if absent), `MONGODB_URI` (point at an Atlas cluster to get native vector search).

## Acceptance checklist
- [ ] Uploading a PDF results in `status:'processed'` with a nonzero `chunkCount` within a reasonable time, or `status:'failed'` with a clear `errorMsg` on a corrupt file.
- [ ] A knowledge-grounded question asked mid-call retrieves and injects relevant chunks (verify via transcript/logs that the system prompt included them).
- [ ] Deleting a `KnowledgeDocument` removes all of its `KBChunk` rows too (no orphans).
- [ ] Retrieval is scoped correctly — an agent never receives chunks from a document that isn't linked to it.
- [ ] A call where the agent says "I don't know" to a real question produces a `KnowledgeGap` row after post-call processing; asking the same question again in a later call increments `count` rather than creating a duplicate.
- [ ] Marking a gap `resolved`, then triggering the same gap again in a new call, resets its status back to `pending`.
