# 05 — Leads & Contacts (Main App CRM)

> Backend domain doc. Source: `backend/src/{models/{Lead,Contact}.model.ts, controllers/{lead,contact}.controller.ts, routes/{lead,contact,contact.public}.routes.ts, services/{lead,contact}.service.ts}`.

> **Scope note**: This file covers only the tenant-facing, main-app CRM features (`Lead` pipeline, `Contact` directory). Other lead-related systems in the codebase (an internal sales-ops panel and its own CRM data model) are out of scope for the current documentation pass and intentionally not covered here.

## 1. Contact (`models/Contact.model.ts`) — CRM directory

The tenant-scoped address book: every callable/emailable person. Distinct from `Lead` (pipeline tracking) — a `Contact` is just "who", a `Lead` is "who + deal stage".

| Field | Type | Notes |
|---|---|---|
| `ownerId` | ObjectId → User | indexed |
| `firstName` | string | required, max 100 |
| `lastName` | string | max 100 |
| `phone` | string | required, E.164 expected |
| `email` / `company` / `jobTitle` | string | optional |
| `status` | enum | `active \| inactive \| blocked` |
| `source` | enum | `manual \| import \| api \| inbound \| campaign` — how the contact entered the system |
| `tags` | string[] | |
| `address` | embedded `{street,city,state,country,zip}` | |
| `linkedinUrl` | string | |
| `lastCalledAt` / `totalCalls` | Date / number | bumped by `call.service.dispatchCall` and `twilio.service.handleInboundCall` every time a `Call` links to this contact |

Indexes: `{ownerId,createdAt:-1}`, `{ownerId,phone:1}`, `{ownerId,status:1}`, plus a text index on `firstName/lastName/company/email` for search.

### API — `/api/contacts` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Paginated list, filters: `search` (regex across name/email/company/phone), `status`, `source` |
| POST | `/` | Create |
| POST | `/import` | CSV upload (multer, 5MB) — parsed via `utils/csv.ts`, then `bulkImportContacts` |
| GET | `/:id` | Detail |
| PATCH | `/:id` | Update |
| DELETE | `/:id` | Delete |

`bulkImportContacts` upserts via `Contact.bulkWrite` keyed on `{ownerId, phone}` with `$setOnInsert` (i.e. **existing contacts with a matching phone are silently skipped, not updated** — re-importing the same CSV is safe/idempotent but won't refresh changed fields on an existing row). Returns `{inserted, matched}` counts.

### Public contact form — `/api/public` (no auth)
`contact.public.routes.ts` is unrelated to the `Contact` CRM model — it's the marketing site's "Contact Us" form:
- `POST /api/public/contact` — Zod-validated (`name`,`email`,`phone?`,`subject`,`message`), saves a `ContactMessage` row, emails the platform admin (address from `PlatformSettings.contactInfo.email`, fallback `support@voxigo.app`) and auto-replies to the sender. Both emails are best-effort (wrapped in their own try/catch so a mail failure never fails the form submission).
- `GET /api/public/contact-info` — returns `PlatformSettings.contactInfo` for the marketing site's footer/contact page.

## 2. Lead — tenant-facing sales pipeline (`models/Lead.model.ts`)

A simple Kanban-style deal tracker each tenant uses for their own prospects — independent from calls/contacts except for optional links.

| Field | Type | Notes |
|---|---|---|
| `ownerId` | ObjectId → User | indexed |
| `contactId` | ObjectId → Contact | optional |
| `name` / `phone` | string | required |
| `email` / `company` | string | optional |
| `value` | number | estimated deal value, USD |
| `stage` | enum | `new → contacted → qualified → proposal → negotiation → won \| lost` |
| `priority` | enum | `low \| medium \| high` |
| `agentId` | ObjectId → Agent | optional — which AI agent is assigned to work this lead |
| `source` | string | free-text, e.g. `'website'`, `'referral'`, `'campaign'` |
| `tags` | string[] | |
| `lastContactedAt` / `nextFollowUpAt` | Date | |
| `emailVerified` | enum | `valid\|invalid\|risky\|catch_all\|unknown` — set by an email-verification integration |

Indexes: `{ownerId,stage:1,createdAt:-1}` (Kanban column queries), `{ownerId,priority:1}`, `{ownerId,nextFollowUpAt:1}` (follow-up reminders), plus a text index on `name/company/email`.

### API — `/api/leads` (JWT)
| Method | Path | Purpose |
|---|---|---|
| GET | `/` | Paginated list; filters `search`, `stage`, `priority`, `agentId`; populates `contactId`/`agentId` |
| GET | `/stats` | Kanban column counts + summed `value` per stage (all 7 stages always returned, zero-filled) |
| POST | `/` | Create — **fires a `lead_created` workflow trigger** (`enqueueWorkflowFollowupsForTrigger`, fire-and-forget) so a workflow can auto-call a new lead; see `06-workflows.md` |
| GET | `/:id` | Detail, populates `contactId`+`agentId` |
| PATCH | `/:id` | Partial update (flat `$set`, no special merge logic needed since there are no nested sub-documents) |
| DELETE | `/:id` | Delete |

## 3. Rebuild Notes (Replit target)

### No billing coupling here
Unlike Agents/Calls/Campaigns, this domain has **no dependency on the Subscription/Plan billing system** — `Lead`/`Contact` are plain tenant-scoped CRUD. Nothing needs to be stripped here for an in-house/no-billing rebuild.

### Dependencies
`mongoose`, `multer` (contact CSV import), `zod` (public contact form only). No other third-party services.
