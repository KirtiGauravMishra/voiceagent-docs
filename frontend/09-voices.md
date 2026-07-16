# 09 — Voices Page

> Frontend domain doc. Source: `frontend/src/pages/VoicesPage.tsx`, `frontend/src/services/voice.service.ts`.

## 1. Overview

`/voices` — a browsable, filterable catalog of every TTS voice available across all 4 providers (OpenAI, Twilio-Polly, Cartesia, ElevenLabs), each with in-browser audio preview. This is a pure discovery/preview tool — it doesn't assign a voice to an agent directly (that happens on `AgentDetailPage`'s Voice tab, `03-agents.md §2`, which duplicates a subset of this same provider/voice-list/preview logic rather than sharing a component).

## 2. Data & providers

`voiceService.list()` → one backend call (`GET /api/voices`) returns all voices in one shot, bucketed as `{openai, polly, cartesia, cartesia_custom, elevenlabs, elevenlabs_custom}` — the `_custom` buckets hold the account's own saved custom voices (Cartesia/ElevenLabs allow custom voice cloning on their platforms; these show up alongside the stock catalog, prefixed with a `✦` marker in `AgentDetailPage`'s equivalent picker per `03-agents.md §2`). All 6 buckets are flattened into one searchable/filterable list client-side.

## 3. Filtering & pagination

Provider filter (quick-select cards, one per provider, each showing its own accent color via `PROVIDER_CFG`), plus dropdown filters for accent and gender (options derived dynamically from whatever values actually appear in the loaded voice list — not a hardcoded enum), a debounced search box (matches name/provider/description), and **client-side pagination** at 12 per page over the filtered set. An "active filter count" badge and a "clear filters" action are shown once any filter is applied.

## 4. Preview playback (`VoiceCard`)

Each card has an editable test-text field (defaulting to `"This is a test text to try the {name} voice."`) and a play button. Preview is only offered for providers with live TTS synthesis available (`openai`, `cartesia`, `elevenlabs` — **not** `twilio-polly`, since Polly voices are rendered natively by Twilio's `<Say>` tag at call time and have no equivalent instant-preview synthesis path in this app). Clicking play calls `voiceService.tts(provider, voiceId, text)`, which:
- POSTs to `/api/voices/tts` expecting a **binary blob** response (`responseType:'blob'`), wraps it in an object URL, and plays it via a plain `<audio>` element.
- Checks a response header `x-preview-note: approximate-openai-fallback` — if present, the requested provider/voice couldn't actually be synthesized and the backend silently substituted an OpenAI voice as an approximation; the UI surfaces this to the user (e.g. "this preview uses an approximate voice") rather than pretending the exact requested voice was heard.
- On error, attempts to parse the blob response body as JSON to extract a human-readable backend error message (since an error response arriving with `responseType:'blob'` doesn't get automatically JSON-parsed by axios — a deliberate error-unwrapping workaround).

Only one voice can play at a time (`playing` state tracks the currently-active voice ID globally on the page).

## 5. Rebuild Notes

### No billing/out-of-scope-panel coupling
Pure main-app feature, no `PlanGate`/billing dependency (note: individual *provider choices* can be plan-gated elsewhere, e.g. ElevenLabs on `AgentDetailPage`'s Voice tab per `03-agents.md §2` — but this discovery/preview page itself shows and previews every provider's voices regardless of plan, since it's not the point where an agent's voice is actually assigned/saved).

### Duplicated preview/provider logic
The TTS-preview mechanics (blob fetch, object URL, approximate-fallback header check, blob-error-unwrapping) and the four-provider voice-list handling are implemented independently on both this page and `AgentDetailPage`'s Voice tab (`03-agents.md §2`). Consolidating into one shared hook/component (e.g. `useVoicePreview()`) during a rebuild would remove this duplication and ensure both surfaces handle the approximate-fallback case identically.

### Dependencies
`lucide-react`. No charting/table library — grid-of-cards layout with plain client-side filter/paginate logic.
