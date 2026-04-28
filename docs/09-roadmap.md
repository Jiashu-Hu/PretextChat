# Roadmap

Each milestone is shippable in isolation; later ones build on earlier ones without retrofitting.

## M0 — Renderer prototype (no extension, no site)

**Goal**: prove pretext + virtualization can scroll a 5000-message synthetic conversation at 60 fps.

- `core/blocks.ts` schema lands (final shape).
- `core/layout.ts` wraps pretext for text and code.
- `core/virtualizer.ts` mounts a viewport.
- A standalone HTML page (`test/playground.html`) feeds in a synthetic patch stream from a fixture.
- No adapter, no extension, no real site.

**Done when**:
- 5000 messages, scroll p99 frame time < 16 ms on a mid-tier laptop.
- Resize from 800 px to 1200 px completes layout in < 50 ms.
- Synthetic streaming (10 patches/frame for 60 frames) doesn't drop a frame.

**Why first**: validates the core thesis. If pretext doesn't deliver, the whole project pivots before adapter work begins.

---

## M1 — ChatGPT read-only viewer

**Goal**: an in-page extension that takes over the ChatGPT message list, read-only.

- `services/markdown.ts`, `services/highlight.ts`, `services/image.ts` complete.
- `services/katex.ts` complete (math is common in ChatGPT).
- `adapters/chatgpt/` complete: fetch interception, parse, no intents yet.
- `shell/` minimal: manifest, content script, page-hook, no options page.
- Renderer activates on conversations > 50 messages; below that, original UI shows.

**Done when**:
- Loading a long ChatGPT conversation shows our renderer with visual fidelity matching the site.
- Code, math, lists, tables, links, images all render acceptably.
- Scrolling and resize are smooth.
- No regressions to login, sidebar, composer.
- Full transcript fallback toggle works.

**Out of scope**: streaming new turns, edit/regenerate, Grok.

---

## M2 — ChatGPT streaming + intents

**Goal**: the active turn streams into our renderer; copy/edit/regenerate work.

- Patch protocol implemented end-to-end: `add`, `append`, `appendLine`, `replace`, `finalize`.
- Markdown state machine in adapter produces patches from SSE chunks.
- Intent protocol implemented: `copy`, `copyMessage`, `edit`, `regenerate`, `feedback`.
- Capability gating wired to renderer chrome.

**Done when**:
- A new prompt streams into our renderer in real time, indistinguishable in latency from the native UI.
- Edit submits via the site's own UI; the resulting patches re-render correctly.
- Regenerate works; branched history is navigable.
- Copy on a code block yields the exact source.

**Out of scope**: Grok.

---

## M3 — Grok adapter

**Goal**: feature parity with M2 on Grok.

- `adapters/grok/` complete.
- Per-site quirks isolated; no renderer changes required.
- Add Grok to manifest matches; activate based on host.

**Done when**:
- All M1 + M2 acceptance criteria pass on Grok.
- `adapters/types.ts` had no breaking changes from M2 → M3 (validates the seam).

---

## M4 — Polish, hardening, docs

**Goal**: ready for public release.

- Options page: per-site enable, theme, debug.
- Popup: quick toggle, status, link to bug report.
- Performance overlay (debug mode): frame times, cache hit rate, mounted-message count.
- Bug-report intent flow: capture sanitized diagnostic + recent patches → user-confirmed open in form.
- Full a11y pass: keyboard nav, screen reader walkthrough, "render full transcript" verified.
- User-facing docs: install, troubleshoot, what to expect when sites update.
- Telemetry **only** if user opts in; off by default; never includes conversation content.

**Done when**:
- One week of dogfooding by 3+ users with no P0 bugs.
- A site change during dogfooding was triaged and fixed in < 1 day.

---

## M5+ (post-v1, deferred)

Listed for context; not committed.

- **Tables as pretext-laid blocks** (currently opaque in v1). Two-pass column layout. Worth it only if profiling shows table-heavy sessions are slow.
- **Block-granularity virtualization** for messages with thousands of blocks. Rare in practice; revisit only if encountered.
- **Generic DOM-scraping adapter** (Strategy B from `04-adapters.md`) for sites without convenient APIs. Useful for Claude.ai, Gemini, etc.
- **In-conversation search** with full-mount (custom find-in-page).
- **Conversation export** as standalone HTML (using the renderer offline).
- **Live tool-call cards** (currently mirrored as static HTML in v1).
- **Mobile browsers** (Firefox Android, Kiwi).

## What is intentionally never on the roadmap

- Sending or syncing conversation data off-device.
- Replacing login or composer.
- Custom AI features layered on top of the chat (we render; we don't summarize, augment, or suggest).
- Cross-site features (e.g., "send this Grok message to ChatGPT") — out of scope and out of remit.
