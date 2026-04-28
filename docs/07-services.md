# Services (`src/services/`)

Leaf utilities consumed by adapters during parsing. They are **not** a third architectural layer — they are libraries the adapter uses to produce blocks. The renderer never imports them.

## Why services live here, not in the renderer

If the renderer imported a syntax highlighter, it would couple to a specific tokenizer, ship the bytes, and constrain future adapters that don't need it. Keeping services adapter-side means:

- The renderer bundle stays small and dependency-free.
- An adapter for a "plain text only" site (e.g., a legacy chat export viewer) doesn't pay the cost.
- Swapping highlighters or KaTeX versions doesn't affect the renderer.

## Module list

### `services/markdown.ts`

Parses markdown to an AST and walks it into `Block[]`. Used by both ChatGPT and Grok adapters via `adapters/shared/`.

```ts
export function markdownToBlocks(
  source: string,
  ctx: { highlight: typeof highlight, katex: typeof katex, image: typeof image }
): Promise<Block[]>
```

- Library: `markdown-it` (smaller, faster than remark for our needs; supports plugins for math fences and tables).
- Plugins enabled: `markdown-it-tex` for `$...$` and `$$...$$`; tables; task lists; strikethrough.
- Returns blocks in document order. Async because services it calls are async (image probe, KaTeX render in v2).

### `services/highlight.ts`

Tokenizes a code source into `Run[][]` (one row per source line).

```ts
export function highlight(
  source: string,
  language: string  // best-effort; '' = no highlighting, returns mono runs
): Run[][]
```

- Library: **Prism** (small, sync, broad language support). Shiki considered and rejected — async API, ships VS Code grammars (large), better quality but worse fit for a layout-time call.
- Output: per line, an array of Runs with `mono: true` and `color: TokenColor` set per token.
- Token color names map from Prism's class names (`token.keyword` → `'keyword'`, etc.). Mapping table lives in this file.
- Languages bundled in v1: ts, tsx, js, jsx, py, go, rs, java, c, cpp, cs, rb, php, sh, sql, json, yaml, html, css, md, diff. ~80% of real-world chat code. Others fall through to `language: ''` (plain mono).

### `services/katex.ts`

Renders LaTeX to HTML and reports intrinsic dimensions.

```ts
export function renderMath(
  source: string,
  display: boolean  // true = display math (block), false = inline
): Promise<{ html: string; width: number; height: number }>
```

- Library: **KaTeX** (faster and smaller than MathJax; sufficient coverage for chat use).
- Renders to a detached div, measures with `getBoundingClientRect`, returns HTML + dims, removes the div.
- Caches by `(source, display)` hash; identical formulas re-rendered for free.
- Used to produce `OpaqueBlock` payloads with `payload.type === 'html'`.

### `services/image.ts`

Probes an image URL for natural dimensions.

```ts
export function probeImage(
  url: string
): Promise<{ width: number; height: number } | null>
```

- Loads via `new Image()`; resolves with `naturalWidth`/`naturalHeight` on `load`.
- Times out at 5 s; returns `null` on failure (adapter can substitute placeholder dims).
- Cache by URL with a small LRU.
- For known site CDN patterns, parses dimensions from URL params first to avoid the round-trip.

### `services/sanitize.ts`

DOMPurify wrapper used before any adapter-emitted HTML lands in an `OpaqueBlock` payload.

```ts
export function sanitizeHtml(html: string): string
```

Adapters are trusted code, but the HTML they emit may include user content (e.g., from a tool-call response). Sanitize at the boundary; cheap and defensive.

## Performance notes

- **Highlighter is the hottest service.** Tokenizing 10k lines on a long conversation load can dominate first-paint. v1 strategy (no viewport coupling required):
  1. The adapter highlights all code blocks in **document order** as it parses messages, but defers tokenization for code blocks beyond the first N messages (configurable; default ~20) via `requestIdleCallback`. The first N messages cover the initial viewport on every realistic conversation length, so this is a good proxy for "what the user is about to look at" without the renderer needing to expose viewport state to the adapter.
  2. Each deferred block emits a `replace` patch once tokenized. Heights don't change (monospace), so no reflow — just repaint.
  3. Viewport-aware prioritization (renderer publishes "currently visible message ids" through a hook the adapter subscribes to) is a deliberate **v2** protocol. It would route through `core/intents.ts` as a renderer-emitted intent, not as a service-to-renderer back-channel; v1 does not need it because document-order deferral already keeps first-paint cheap on the conversations we target.

- **KaTeX is moderate.** A typical math-heavy message has 5–20 formulas. Render synchronously during initial parse; the cost is bounded. Cache aggressively.

- **Image probe is async by nature.** During initial parse, emit the image block with placeholder dims if probe is slow; emit `replace` once dims arrive.

## Lazy-loading services

Services are **dynamically imported** by the adapter on first use:

```ts
// adapters/shared/markdown.ts
const highlight = (await import('../../services/highlight.js')).highlight
const katex     = (await import('../../services/katex.js')).renderMath
```

This keeps the initial content-script payload small. KaTeX (~250 KB) and the highlighter (~150 KB) only load when the conversation contains math or code.

## Testing

- **Highlight golden tests**: known source + language → expected `Run[][]`. Lock down regressions when Prism updates.
- **KaTeX dimension tests**: a small set of formulas with hand-measured dimensions; assert within ±1 px.
- **Markdown round-trip tests**: source → blocks → render to plain text, compare against expected output. Catches AST traversal bugs.
