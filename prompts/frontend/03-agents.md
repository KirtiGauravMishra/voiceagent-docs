# Frontend Prompt 03 — Agents Pages

> Read `../00-master-rebuild-guide.md`, `../backend/02-agents.md`, `../backend/03-calls-realtime-pipeline.md`, `../backend/07-knowledge-base.md`, `../backend/08-notifications-settings.md` first.

## Goal
`AgentsPage` (`/agents`, list + create/clone/delete) and `AgentDetailPage` (`/agents/:id`, the full configuration editor).

## `AgentsPage` — list view

Fetch agents (a single reasonably-sized page is fine — add real pagination once an account's agent count outgrows one page), client-side-filtered by a search box matching `name`/`description`.

### `AgentCard` grid
Each card: name, active/inactive status dot, description, and a hover action row (Make Call, Settings → `/agents/:id`, a "⋮" menu with Clone/Delete). Don't add per-agent call-stat tiles (calls handled, success rate, avg duration) unless you're wiring them to a real backend aggregation — a hardcoded placeholder stat is worse than no stat at all.

### Modals
- **`CreateAgentModal`** — four creation modes via a segmented control:
  - **Auto Build** (default) — user fills in goal/next-steps/FAQs/sample-conversation as plain text fields; assemble these client-side into a structured system-prompt string (template assembly, no LLM call needed for this step).
  - **From Scratch** — a raw textarea for a fully custom system prompt.
  - **Inbound Calls** / **Outbound Calls** — an industry-template picker (a static local JSON data file with categories/templates using `{agent_name}`/`{business_name}`/etc. placeholders, filled in client-side). Outbound mode additionally collects campaign type/product/target-audience/call-goal fields appended to the prompt.
  - On submit: send `{name, systemPrompt, language}` — leave every other field (voice, callSettings, taskSettings) at backend defaults, configured later on the detail page.
- **`CloneAgentModal`** — fetch the full source agent, copy every configurable field (`systemPrompt`, `description`, `language`, `status`, `llm`, `voice`, `callSettings`, `taskSettings`, `knowledgeDocIds`) into a new create call with a "(Copy)"-suffixed name — a real deep clone.
- **`DeleteAgentModal`** — require typing the exact agent name to confirm, since deletion is irreversible. If the backend rejects the delete because the agent is still assigned to a running campaign, an active workflow, or a phone number's inbound routing (Prompt `backend/02-agents.md`), surface that exact message instead of a generic failure toast — the user needs to know what to unassign first.
- **`MakeCallModal`** — country-code + phone-number quick-dial form, load the account's phone numbers for an optional "from number" override, dispatch via the calls API (`{agentId, to, direction:'outbound'}`).

Build Share/Clone/Delete/MakeCall as **shared components** (e.g. under `components/agents/`), reused by both `AgentsPage` and `AgentDetailPage` — don't duplicate the same modal in both files.

## `AgentDetailPage` — the configuration editor

One tab strip + one "Save Changes" action that submits **every** editable field at once, not per-tab saves. Lift all tab state into the parent component, load once on mount, write back with one update call.

On load, if the agent's `voice.provider` isn't one of your currently-supported providers, coerce it to a safe default (`openai`) rather than rendering a broken/unselected dropdown — a defensive guard for data written under a since-changed provider list.

### Tabs

**Agent Details** — name is edited inline in the page header (an auto-sizing input that grows with name length); this tab holds the system-prompt textarea and basic identity fields.

**LLM** — a combined `provider::model` dropdown (OpenAI and Anthropic model options — keep this list current with what those providers actually offer; expect to update it periodically, or consider a free-text model field instead of a maintained dropdown to avoid the staleness problem entirely), max-tokens input (50–4096), temperature slider (0–2). Also host the **Knowledge Base attachment** picker — a checkbox list of the account's `ready`-status knowledge documents (Prompt `backend/07-knowledge-base.md`).

**Voice** — provider dropdown (OpenAI / Cartesia / ElevenLabs), then a voice picker whose options depend on the selected provider:
- OpenAI: a fixed small set of named voices (alloy/echo/fable/onyx/nova/shimmer or your provider's current list).
- Cartesia/ElevenLabs: fetch the live voice list from your backend's voices endpoint; if empty, fall back to a curated hardcoded list so the picker is never empty.
- A live audio-preview button synthesizes a short sample line and plays it in-browser (object URL + `<audio>`), resetting whenever the selected voice changes.
- A speed slider (0.5–2.0).

**Call** — max duration slider (30–300s, hard-capped at 300 client-side), greeting message, a "thinking placeholder" field with a few quick-fill preset buttons, recording/transcription toggles, a hangup-prompt textarea with an "Auto Fill" example-preset button, farewell message, a full **Human Handoff** section (enable toggle → transfer phone number with E.164 validation feedback, handoff message, plain-English transfer-conditions textarea with quick-add preset chips, and a live "test the transfer" action that calls the backend's real bridge-call test-transfer endpoint), voicemail-detection/retry toggles, max-retries slider.

**Task** — summarization/extraction/success-evaluation toggles and their prompt fields (extraction prompt, webhook URL) — a direct mirror of the backend's `taskSettings` shape.

**Tools** — Cal.com appointment-booking configuration: check whether Cal.com is connected, show a connect/connected banner accordingly, expose optional event-type-ID/timezone override fields (leave blank to let the backend auto-resolve from the connected account). Only build a custom-tools list here if you're actually wiring it to backend function-calling — don't ship a fake "add a tool" UI with no persistence.

## Header actions
Clone / Delete buttons (shared components, per above), plus an agent-ID copy-to-clipboard affordance next to the inline name editor.

## Dependencies
`lucide-react`, `react-hot-toast`. Plain controlled `useState` inputs are fine for this page's forms — no form-library requirement here.

## Acceptance checklist
- [ ] Creating an agent via each of the four creation modes produces a working agent with a sensible system prompt.
- [ ] Cloning an agent copies every configurable field, not just name/prompt.
- [ ] Editing any single field in any tab and clicking Save Changes persists that change without clobbering fields in other tabs.
- [ ] The voice preview plays the correct voice for the currently-selected provider/voice combination.
- [ ] The Human Handoff "test transfer" action places a real call and bridges to the configured number.
- [ ] Deleting an agent requires typing its exact name and is irreversible only after that confirmation.
