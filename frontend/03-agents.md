# 03 — Agents Pages

> Frontend domain doc. Source: `frontend/src/pages/{AgentsPage,AgentDetailPage}.tsx`, `frontend/src/hooks/useAgents.ts`, `frontend/src/services/agent.service.ts`.

> **Scope note**: `AgentDetailPage` has two tabs (`AI Voice Widget`, `AI Chat Widget`) tied to the embeddable public widget feature, which is out of scope per current project priorities. Mentioned only briefly in §4; not detailed.

## 1. `AgentsPage` (`/agents`) — list view

Fetches all agents via `agentService.getAll(1, 100)` on mount (a flat 100-item page, no real pagination UI — the list view assumes an account won't have more than 100 agents), client-side filtered by a search box matching `name`/`description`. Each agent is "adapted" (`adaptAgent()`) from the raw API shape into a display-only type carrying placeholder stat fields (`callsHandled: 0, successRate: 0, avgDuration: '—', model: 'gpt-4o-mini'`) that **are never populated from real data** — these are static/hardcoded placeholders in the current implementation, not wired to any backend call-stats aggregation. Worth flagging clearly: if the rebuild wants real per-agent stats on this card, that's new work, not a bug fix.

### Grid of `AgentCard`s
Each card shows: name, active/inactive status dot, description, the four (placeholder) stat tiles, and a hover action row (Make Call, Settings → navigates to `/agents/:id`, and a "⋮" menu with Share/Clone/Delete).

### Modals (all rendered via `createPortal` into `document.body`, each a self-contained component in this same file)
- **`CreateAgentModal`** — the most complex piece of this page. Four creation "modes," selected via a segmented control:
  - **Auto Build** (default) — user describes goal/next-steps/FAQs/sample-conversation in plain text; `buildSystemPrompt()` assembles these into a structured markdown-ish system prompt client-side (no AI call involved despite the "Auto Build" name — it's template assembly, not LLM-generated).
  - **From Scratch** — a raw textarea for a fully custom system prompt.
  - **Inbound Calls** / **Outbound Calls** — an **industry template picker** (`agentPromptsData.categories`, a static local JSON file — `frontend/src/data/agentPrompts.json` — not fetched from the backend) with `{agent_name}`/`{business_name}`/etc. placeholders filled in via `fillTemplate()`. The industry-template grid itself is wrapped in `<PlanGate feature="autoPromptTemplates">` (billing-plan-gated — see `12-shared-components-state-styling.md` for `PlanGate`); Outbound mode additionally collects campaign type/product/target-audience/call-goal fields appended to the prompt.
  - The **Outbound mode button itself** is disabled (grayed out, "Upgrade your plan to use") unless `isFeatureEnabled('outboundCalls')` — same billing-plan-feature-flag pattern used throughout the app (`useSubscription()`, see `12-shared-components-state-styling.md`).
  - On submit: only sends `{name, systemPrompt, language}` to `agentService.create()` — every other Agent field (voice, callSettings, taskSettings, etc.) is left at whatever the backend schema defaults to, configured later on the detail page.
- **`CloneAgentModal`** — fetches the full source agent (`agentService.getById`), copies every configurable field (`systemPrompt`, `description`, `language`, `status`, `llm`, `voice`, `callSettings`, `taskSettings`, `knowledgeDocIds`) into a new `create()` call with a "(Copy)"-suffixed name — a real deep-clone, not just a name/prompt copy.
- **`DeleteAgentModal`** — requires typing the exact agent name to confirm (a stronger-than-usual confirmation pattern, appropriate given deletion is irreversible on the backend).
- **`ShareAgentModal`** — generates a `/share/agent/:id` link and copies it to clipboard, with copy stating "view-only, expires in 7 days." **This is UI-only** — there is no corresponding `/share/agent/:id` route registered in `App.tsx` and no backend endpoint for a shareable/expiring view-only agent link found in the backend docs. This feature appears to be a stubbed/unfinished UI affordance; clicking through the generated link would 404. Flag for a decision during rebuild: implement the real feature, or remove this modal.
- **`MakeCallModal`** — a country-code + phone-number quick-dial form, loads the account's phone numbers (`phoneNumberService.list()`) for an optional "from number" override, and calls `callService.dispatch({agentId, to, direction:'outbound'})`. Gated behind `isFeatureEnabled('outboundCalls')` the same way as the Outbound creation mode.

## 2. `AgentDetailPage` (`/agents/:id`) — the configuration editor

A single page with a tab strip and one "Save Changes" action that submits **every** editable field at once (not per-tab saves) — all tab state is lifted into the parent component (`agentName`, `systemPrompt`, `voiceSettings`, `callSettings`, `taskSettings`, `llmConfig`, `kbDocIds`, `voiceWidget`, `calcomEventTypeId`, `calcomTimeZone`), loaded once from `agentService.getById()` on mount and written back with one `agentService.update()` call from a single `handleSave()`.

One notable load-time migration: if the loaded agent's `voice.provider` isn't one of the four currently-valid values, it's silently coerced to `openai` — a defensive guard against legacy data from a since-removed voice provider (`'elevenlabs'` is explicitly called out in a comment as a provider that was at one point removed then reinstated, i.e. the schema has churned over time).

### Tabs

**Agent Details** — name is actually edited inline in the page header (not in this tab — an auto-sizing text input that grows with the name length), so this tab holds the system prompt textarea and basic identity fields.

**LLM** — a single combined `provider::model` dropdown (4 hardcoded options: OpenAI GPT-4.1 Mini/GPT-4.1, Anthropic Claude Haiku 4.5/Sonnet 4.5 — model IDs are hardcoded in this component with a comment noting they were last verified against Anthropic's API in "March 2026" and that two older Claude models are EOL and were removed — **this hardcoded list will go stale over time and needs manual updates** as providers ship new models), a max-tokens number input (50–4096), and a temperature slider (0–2). Also hosts the **Knowledge Base attachment** picker — a checkbox list of the account's `processed`-status documents (fetched live from `knowledge.service`), wrapped in `<PlanGate feature="knowledgeBase">`.

**Voice** — provider dropdown (OpenAI / Cartesia / ElevenLabs-if-plan-allows-or-already-set), then a voice picker whose options depend on the selected provider:
- OpenAI: 6 hardcoded voices (alloy/echo/fable/onyx/nova/shimmer).
- Twilio-Polly: tries a live API list (`voiceService.list()`) first, falls back to a hardcoded 7-voice list if the API list is empty.
- Cartesia: same live-list-with-fallback pattern, falling back to a **large hardcoded list of ~29 named voices** (with per-voice gender/accent/language descriptions, including a dedicated Hindi/Indian voice section) if the API doesn't return anything — this fallback list is effectively a manually-curated Cartesia voice catalog baked into the frontend.
- ElevenLabs: API-list only, no hardcoded fallback.
- A **live audio preview** button synthesizes "Hello! This is a preview of the selected voice." via `voiceService.tts()` and plays it in-browser (object URL + `<audio>`), auto-stopping/resetting whenever the selected voice changes.
- A speed slider (0.5–2.0).

**Call** (wrapped in `<PlanGate feature="outboundCalls">` — the entire tab is billing-gated) — the largest tab: max duration slider (30–300s, hard-capped at 300 client-side regardless of what's stored), greeting message, a "thinking placeholder" field with 4 quick-fill preset buttons, recording/transcription toggles, a hangup-prompt textarea with an "Auto Fill" button that inserts a full example hangup-rules paragraph, a farewell-message field, a full **Human Handoff** section (enable toggle → transfer phone number with E.164 validation feedback, handoff message, plain-English transfer-conditions textarea with quick-add preset chips, and a **live "test the transfer" widget** that calls `agentService.testTransfer(agentId, testNumber)` — a real bridge call placed via the backend, see `../backend/02-agents.md §4` and `../backend/03-calls-realtime-pipeline.md`), voicemail-detection/retry toggles, max-retries slider, and a read-only "Max Concurrent Calls" display sourced from the account's plan (billing-derived, not agent-specific).

**Task** — summarization/extraction/success-evaluation toggles and their associated prompt fields (extraction prompt, webhook URL) — a fairly direct 1:1 mirror of the backend's `taskSettings` shape (`../backend/02-agents.md`).

**Tools** — Cal.com appointment-booking configuration: checks whether Cal.com is connected (via `settings.service.listProviders`/`listIntegrations` — checking **both** catalogs, since Cal.com historically appears in both the Providers and Integrations lists on the backend, see `../backend/08-notifications-settings.md §3`), shows a connect/connected banner accordingly, and exposes optional event-type-ID/timezone override fields (leaving both blank lets the backend auto-resolve from the connected Cal.com account — see `../backend/03-calls-realtime-pipeline.md` for how this feeds the live-call Cal.com tool-calling). Below that, a **"Future custom tools" section is explicitly a non-functional placeholder** — it lets the user add/remove function names (`get_calendar_availability`, `book_appointment`, `send_sms`, etc.) to a local list with no persistence and no backend wiring; the component's own copy states "Additional tools below are not wired yet." Only the Cal.com booking tools (get_calendar_availability/book_appointment/cancel_appointment) actually function, and only for OpenAI models per the in-component note.

## 3. Header actions

Clone / Share / Delete buttons (reusing the same modals as the list page, imported/duplicated logic — note: `AgentsPage` and `AgentDetailPage` each define their **own copies** of `ShareAgentModal`/`CloneAgentModal`/`DeleteAgentModal`/`MakeCallModal` rather than sharing one component — a consolidation opportunity for a rebuild, since the two implementations are largely identical), plus the agent-ID copy-to-clipboard affordance next to the inline name editor.

## 4. Out-of-scope tabs (not detailed)

`AI Voice Widget` and `AI Chat Widget` tabs configure the embeddable third-party widget (icon upload, primary color, position, call restrictions, chat widget code snippet) — tied entirely to the public widget feature excluded from this documentation pass. If the public widget is dropped from the rebuild, these two tabs (and the `voiceWidget` state/field plumbing threaded through `AgentDetailPage`) can be removed along with them.

## 5. Rebuild Notes

### Billing coupling
`PlanGate`-wrapped sections (Outbound-mode creation, Knowledge Base attachment, the entire Call tab, industry template picker) and `isFeatureEnabled()` checks (Outbound calling in the create-modal and make-call-modal) are all billing-plan-driven. If billing is stripped from the rebuild, decide on a blanket policy (e.g. treat every feature flag as always-enabled) rather than leaving `PlanGate`/`useSubscription` calls pointed at a removed backend — see `12-shared-components-state-styling.md` for the shared `PlanGate` component and how to simplify/remove it.

### Known incomplete/placeholder features to resolve before rebuild
- **Share Agent link** — generates a link to a route that doesn't exist (§1). Implement or remove.
- **Custom Tools list** (Tools tab) — explicitly non-functional placeholder (§2). Implement or remove.
- **AgentCard stat placeholders** (`callsHandled`, `successRate`, `avgDuration`, `lastActive` beyond creation date) — currently hardcoded zeros/dashes, not backed by any aggregation query. Either wire to a real per-agent stats endpoint or remove the tiles.
- **Hardcoded LLM model list** — will need periodic manual updates as OpenAI/Anthropic ship new models; consider whether this should instead be a simple text input (trusting the backend's free-text `model` field) rather than a maintained dropdown.

### Duplicated modal components
`AgentsPage.tsx` and `AgentDetailPage.tsx` each independently define equivalent Share/Clone/Delete/MakeCall modal components — consolidate into shared components under `components/agents/` during a rebuild to avoid double-maintenance.

### Dependencies
`lucide-react`, `react-hot-toast`, no form-library usage in these two files specifically (plain controlled `useState` inputs throughout, unlike the auth pages which use `react-hook-form`).
