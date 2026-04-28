# chatPlugin — design docs

A browser extension that replaces the message list in ChatGPT and Grok with a fast renderer built on [`chenglou/pretext`](https://github.com/chenglou/pretext) + viewport virtualization. Aims to keep long conversations smooth.

## Read in this order

| # | Doc | What it covers |
|---|---|---|
| 00 | [overview](00-overview.md) | Problem, goals, non-goals, success criteria |
| 01 | [architecture](01-architecture.md) | Two-layer split, folder layout, dependency rules |
| 02 | [block-schema](02-block-schema.md) | The contract between renderer and adapters |
| 03 | [renderer](03-renderer.md) | Layer 1: pretext + virtualization (universal) |
| 04 | [adapters](04-adapters.md) | Layer 2: per-site parsing and intent handling |
| 05 | [streaming](05-streaming.md) | Patch protocol for live turns and edits |
| 06 | [intents](06-intents.md) | Action protocol — declarative, capability-gated |
| 07 | [services](07-services.md) | Markdown / highlight / KaTeX / image utilities |
| 08 | [extension-shell](08-extension-shell.md) | MV3 manifest, content script, page hook |
| 09 | [roadmap](09-roadmap.md) | M0 prototype → M4 release |
| 10 | [risks](10-risks.md) | Risk register with mitigations |
| ★  | [CONTRACTS](CONTRACTS.md) | Index of cross-cutting types (single source of truth) |

## Architecture at a glance

```
shell  →  adapter (per-site)  →  services (highlight, katex, image)
shell  →  renderer (universal)
adapter ──── Block schema ────→ renderer
```

- **Renderer** is site-agnostic, knows pretext, never imports adapters or services.
- **Adapters** are per-site, parse network/DOM into `Block`s, handle `Intent`s.
- **Services** are leaf utilities the adapter calls during parsing.
- **Block schema** is the only thing crossing the seam.

## What's decided vs. open

**Decided** (don't relitigate without new evidence):
- Two-layer architecture (renderer + adapter).
- Pretext is the layout engine for text and code; LaTeX stays opaque.
- Tables stay opaque in v1 (revisit only if profiling demands it).
- Network interception preferred over DOM scraping per adapter.
- MV3 extension; ChatGPT and Grok in v1.
- Patch protocol for streaming; intent protocol for actions.

**Open** (decide during implementation):
- Final pretext API surface we depend on (locked when M0 ships).
- Highlighter choice — Prism leading; revisit if Prism quality issues hit.
- Whether to detach the original site DOM in v2 (keep mounted in v1).
- Telemetry shape (opt-in only; details TBD).

## Status

Planning. No code yet.
