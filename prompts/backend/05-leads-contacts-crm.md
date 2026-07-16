# Backend Prompt 05 — Leads & Contacts

> Read `../00-master-rebuild-guide.md` first.

## Goal
Two simple CRUD domains: a `Contact` directory (anyone callable/emailable) and a `Lead` sales pipeline (Kanban-style deal tracker). These are independent of each other except for an optional link.

## Data model — `Contact`

`ownerId`, `firstName` (required, max 100), `lastName` (max 100), `phone` (required, E.164), `email`, `company`, `jobTitle`, `status` (enum `active|inactive|blocked`), `source` (enum `manual|import|api|inbound|campaign`), `tags` (string[]), `address` (embedded `{street,city,state,country,zip}`), `linkedinUrl`, `lastCalledAt`, `totalCalls` (bump this + `lastCalledAt` whenever a `Call` links to this contact — from Prompt 03 and Prompt 04).

Indexes: `{ownerId,createdAt:-1}`, `{ownerId,phone}`, `{ownerId,status}`, text index on `firstName/lastName/company/email`.

### API — `/api/contacts` (JWT)
`GET /` (paginate, filter `search`/`status`/`source`), `POST /`, `POST /import` (CSV upload, multer ≤5MB), `GET/PATCH/DELETE /:id`.

CSV import: upsert by `{ownerId, phone}` — an existing contact with a matching phone is left untouched (`$setOnInsert`), not updated; re-importing the same file is safe/idempotent. Return `{inserted, matched}` counts.

## Data model — `Lead`

`ownerId`, `contactId` (optional →Contact), `name`/`phone` (required), `email`, `company`, `value` (number, estimated deal value), `stage` (enum `new→contacted→qualified→proposal→negotiation→won|lost`), `priority` (enum `low|medium|high`), `agentId` (optional →Agent), `source` (free text), `tags`, `lastContactedAt`/`nextFollowUpAt`, `emailVerified` (enum `valid|invalid|risky|catch_all|unknown` — only populate this if you build an email-verification integration; otherwise omit the field).

Indexes: `{ownerId,stage,createdAt:-1}`, `{ownerId,priority}`, `{ownerId,nextFollowUpAt}`, text index on `name/company/email`.

### API — `/api/leads` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | paginated, filters `search`/`stage`/`priority`/`agentId` |
| GET | `/stats` | counts + summed `value` per stage — always return all 7 stages, zero-filled for empty ones (Kanban column headers) |
| POST | `/` | create — fire a `lead_created` workflow trigger (Prompt 06) if you're building workflows |
| GET | `/:id` | detail |
| PATCH | `/:id` | partial update |
| DELETE | `/:id` | delete |

## Environment variables
None specific to this module.

## Acceptance checklist
- [ ] Re-uploading the same contacts CSV twice does not create duplicates and does not clobber manually-edited fields on existing contacts.
- [ ] `GET /api/leads/stats` always returns exactly 7 stage buckets even when some have zero leads.
- [ ] Creating a lead fires a `lead_created` trigger if the Workflows module is present (safe no-op otherwise).
