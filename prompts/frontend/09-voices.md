# Frontend Prompt 09 — Voices Page

> Read `../00-master-rebuild-guide.md`, `../backend/02-agents.md`, `03-agents.md` first.

## Goal
`/voices` — a browsable, filterable catalog of every TTS voice available across all supported providers (OpenAI, Cartesia, ElevenLabs), each with in-browser audio preview. A pure discovery/preview tool — it doesn't assign a voice to an agent directly (that happens on `AgentDetailPage`'s Voice tab, Prompt `frontend/03-agents.md`).

## Data & providers
One backend call returns all voices bucketed by provider (`openai`, `cartesia`, `cartesia_custom`, `elevenlabs`, `elevenlabs_custom` — the `_custom` buckets hold the account's own saved custom/cloned voices on Cartesia/ElevenLabs, shown alongside the stock catalog with a distinct marker, e.g. a `✦` prefix). Flatten all buckets into one searchable/filterable list client-side.

## Filtering & pagination
Provider filter (quick-select cards, one per provider, each with its own accent color), dropdown filters for accent and gender (derive the available options dynamically from whatever values actually appear in the loaded voice list, not a hardcoded enum), a debounced search box (matches name/provider/description), and client-side pagination (e.g. 12/page) over the filtered set. Show an "active filter count" badge and a "clear filters" action once any filter is applied.

## Preview playback (`VoiceCard`)

Each card: an editable test-text field (default something like `"This is a test text to try the {name} voice."`) and a play button. Build this as a **shared `useVoicePreview()` hook** so this page and `AgentDetailPage`'s Voice tab don't each reimplement the same preview mechanics independently.

The hook should:
- POST to the TTS-preview endpoint expecting a binary blob response, wrap it in an object URL, and play via a plain `<audio>` element.
- Check for an "approximate fallback" response header — if present, the requested provider/voice couldn't actually be synthesized and the backend substituted a fallback voice as an approximation; surface this to the user (e.g. "this preview uses an approximate voice") rather than implying the exact requested voice was heard.
- On error, parse the blob response body as JSON to extract a human-readable backend error message (a blob-typed error response isn't auto-JSON-parsed by most HTTP clients — unwrap it explicitly).
- Track a single globally-playing voice ID so only one preview plays at a time (starting a new preview stops any currently-playing one).

## Dependencies
`lucide-react`. No charting/table library — a grid-of-cards layout with client-side filter/paginate logic.

## Acceptance checklist
- [ ] Every provider's voices are browsable and filterable by accent/gender/search without a page reload.
- [ ] Clicking play on a voice card plays that exact voice (or clearly indicates when an approximate fallback was used instead).
- [ ] Starting a new preview stops any currently-playing one — never two previews audible simultaneously.
- [ ] The same preview hook/logic is reused (not duplicated) between this page and the agent detail page's Voice tab.
- [ ] Custom/cloned voices (Cartesia/ElevenLabs) appear alongside the stock catalog with a clear visual distinction.
