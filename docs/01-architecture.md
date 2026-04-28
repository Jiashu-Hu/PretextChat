# Architecture

## Layered split

```
┌──────────────────────────────────────────────────────────────┐
│  Shell (extension lifecycle)                                 │
│   - manifest, content-script entry, host detection           │
│   - picks adapter, instantiates renderer, wires intents      │
└──────────────────────────────────────────────────────────────┘
            │  selects                       │  mounts
            ▼                                ▼
┌─────────────────────────────┐   ┌──────────────────────────┐
│  Layer 2: Adapters          │   │  Layer 1: Renderer       │
│  (per-site, breakable)      │──▶│  (universal, stable)     │
│   - fetch / intercept       │   │   - pretext layout       │
│   - parse markdown → blocks │   │   - virtualization       │
│   - tokenize code           │   │   - opaque mounting      │
│   - render LaTeX → opaque   │   │   - height cache         │
│   - probe images            │   │   - scroll mgmt          │
│   - handle intents          │   │   - emits intents        │
└─────────────────────────────┘   └──────────────────────────┘
            │                                ▲
            │  emits Message / MessagePatch  │  via Block schema
            └────────────────────────────────┘
                       (the contract)
```

The **Block schema** (see `02-block-schema.md`) is the load-bearing contract. Both layers depend on it; nothing else crosses the seam.

## Why two layers and not one

- **Sites change; pretext does not.** When ChatGPT ships a new conversation API or markup change, only the adapter needs work. The renderer is unaffected.
- **The renderer is testable in isolation.** Synthetic block streams replay any conversation shape without a browser or a network.
- **New site = new adapter.** Adding Claude.ai, Gemini, Perplexity, Mistral chat, etc. is a folder under `adapters/` plus a registration line — not a fork.

## Why not three layers (extracting "shared services" as their own layer)

Highlighting, KaTeX, and image probing are **utilities the adapter calls during parsing**, not a distinct architectural layer. Treating them as a layer would imply the renderer depends on them — defeating the universality goal. They live in `services/` as plain modules consumed by adapters.

## Folder layout

```
chatPlugin/
  docs/                      ← this directory
  src/
    core/                    ← Layer 1: renderer (no site code, no parsers)
      blocks.ts              block schema, patches, BLOCK_SCHEMA_VERSION
      adapter.ts             Adapter interface, Capability union (the seam)
      intents.ts             Intent / IntentResult types
      layout.ts              pretext wrapper, height cache
      virtualizer.ts         windowed mount/unmount
      mount.ts               opaque block hosting + ResizeObserver
      runtime.ts             scroll container, viewport tracking
      theme.ts               token-color → hex resolver (light/dark)
      index.ts               public API: createRenderer({ adapter })
    services/                ← shared utilities, called by adapters
      highlight.ts           source → Run[][]
      katex.ts               latex → { html, width, height }
      image.ts               url → { width, height }
      markdown.ts            md AST → Block[]   (used by both adapters)
    adapters/                ← Layer 2: per-site
      shared/                ← cross-site building blocks (markdown pipeline, etc.)
                             (the Adapter interface lives in core/adapter.ts;
                              adapters import it, not the other way around)
      chatgpt/
        index.ts             registers adapter
        fetch.ts             intercept conversation endpoint
        parse.ts             site payload → Block[]
        controls.ts          intent → site UI action
        selectors.ts         DOM selectors / capability detection
      grok/
        index.ts
        fetch.ts
        parse.ts
        controls.ts
        selectors.ts
    shell/                   ← extension lifecycle
      content.ts             content-script entry; picks adapter, mounts renderer
      background.ts          MV3 service worker (minimal: install, options page)
      options.html           per-site enable, debug toggles
      manifest.json
  test/
    fixtures/                synthetic conversations (5k messages, code-heavy, math-heavy, mixed)
    renderer/                renders fixtures headless
    adapters/                snapshot tests on captured payloads
  package.json
  tsconfig.json
  README.md
```

## Direction of dependency

```
shell  →  adapter  →  services
shell  →  renderer
adapter  → blocks (types only)
renderer → blocks (types only)
services → blocks (types only)
```

- The renderer **never** imports any module under `adapters/` or `services/`. It does depend on the `Adapter` *interface*, but that interface lives in `core/adapter.ts` — defined by the host, implemented by adapters. This is the standard ports-and-adapters pattern: both layers depend on a contract owned by the host, not on each other.
- Adapters import only from `core/blocks.ts`, `core/adapter.ts`, and `core/intents.ts` (the seam types). They never import `createRenderer`, the renderer runtime, or any services they don't use.
- Services may import schema **types** from `core/blocks.ts` (so `markdown.ts` can return `Block[]` and `highlight.ts` can return `Run[][]`) but **never** import core runtime (renderer, virtualizer, mount, theme, etc.) or any `adapters/` module.
- The shell is the only place that imports `createRenderer` and a concrete `Adapter` implementation and wires them together.

This direction is enforced by an ESLint boundary rule (`eslint-plugin-boundaries` or equivalent) so violations fail CI. The `core/blocks.ts` module is the schema's single source of truth — types plus the `BLOCK_SCHEMA_VERSION` constant only, no behavior — so depending on it from services creates no behavioral coupling to the renderer.

## Stability claim — calibrated

A clean seam does not prevent breakage; it **localizes** it. When ChatGPT changes its conversation payload shape, the failure surface is `adapters/chatgpt/fetch.ts` and `parse.ts`. The renderer, the schema, the other adapter, and the shell are unaffected. Expected fix time: hours, not days.

What two layers do not buy us:
- Immunity to site updates (still happens).
- Immunity to pretext API changes (would touch `core/layout.ts` only, but still touches us).
- Immunity to browser changes (Chrome MV3 deprecations, etc. — affects shell).

## Build and bundling

- Single TypeScript project, ESM, bundled with esbuild or Vite.
- **Three** output bundles, matching the manifest:
  - `content.js` — ISOLATED-world content script. Renderer (`core/`), services (`services/`), and both adapters (`adapters/`) ship here. The adapter is chosen at runtime via host match; no per-site bundle split in v1 (size is small enough; revisit if it grows).
  - `page-hook.js` — MAIN-world content script. Contains only the fetch monkey-patch, the bootstrap rendezvous, and the early-stream pump. Must be a separate bundle because MAIN- and ISOLATED-world scripts run in different JS realms; folding it into `content.js` would put the monkey-patch in the wrong world.
  - `background.js` — MV3 service worker. Storage helpers, options page open, install/update lifecycle. No conversation data passes through.
- Pretext is bundled directly. KaTeX and the highlighter are dynamically imported so adapters that don't need them do not pay the cost (e.g., a future "plain text only" adapter).
