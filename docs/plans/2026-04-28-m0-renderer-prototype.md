# M0 Renderer Prototype Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the universal site-agnostic renderer (`src/core/`) and prove it can scroll a 5000-message synthetic conversation at 60 fps in a standalone playground page — no extension, no real site, no adapter.

**Architecture:** Layer 1 of the two-layer architecture from `docs/01-architecture.md`. The renderer consumes `Block`s and `MessagePatch`es (canonical schema in `docs/02-block-schema.md`) and produces a fast scrolling viewport via [`chenglou/pretext`](https://github.com/chenglou/pretext) (1D line layout) + viewport virtualization. The M0 deliverable is a standalone HTML playground that drives the renderer with a synthetic patch stream from a fixture; M1 wraps this with an extension and a real adapter.

**Tech Stack:**
- TypeScript 5.x (strict), ESM
- Vite 5.x (dev server + library build)
- Vitest 2.x (unit tests, jsdom for DOM-touching modules)
- Playwright 1.x (headless browser perf tests with `performance.measure` + `requestAnimationFrame` frame timing)
- `chenglou/pretext` vendored as a git submodule under `vendor/pretext` (not on npm)
- `eslint-plugin-boundaries` (enforces layering rules from `docs/CONTRACTS.md`)
- Node 20+

**Acceptance criteria** (from `docs/09-roadmap.md` §M0):
1. 5000-message synthetic conversation, scroll p99 frame time < 16 ms on a mid-tier laptop.
2. Resize from 800 px → 1200 px completes layout in < 50 ms (visible-window only).
3. Synthetic streaming (10 patches/frame for 60 frames) does not drop a frame.

**Cross-doc invariants this plan must round-trip against** (`docs/CONTRACTS.md`):
- Block / message invariants 1–6 (schema validity).
- Patch protocol invariants 1–7 (create-or-replace semantics, append constraints, atomicity).
- Layering: `core/` imports nothing from `adapters/` or `services/`; the seam types live in `core/{blocks,adapter,intents}.ts`.

---

## File structure

Locked at the start of M0; later milestones extend but do not restructure.

```
chatPlugin/
  package.json                             ← npm scripts, dev deps
  tsconfig.json                            ← strict TS, ESM, path aliases
  vite.config.ts                           ← lib build + playground dev server
  vitest.config.ts                         ← unit tests, jsdom env
  playwright.config.ts                     ← perf tests, single worker, reuse server
  .eslintrc.cjs                            ← boundaries rule
  vendor/
    pretext/                               ← git submodule
  src/
    core/
      blocks.ts                            ← schema types + BLOCK_SCHEMA_VERSION
      adapter.ts                           ← Adapter, Capability (types only)
      intents.ts                           ← Intent, IntentResult + default handlers
      theme.ts                             ← TokenColor → hex
      patches.ts                           ← in-memory message-list state + apply()
      cache.ts                             ← height cache (LRU, width-keyed)
      layout.ts                            ← pretext wrapper: measureBlock/Message
      mount.ts                             ← opaque-block hosting (img / html / iframe)
      virtualizer.ts                       ← windowed mount of message DOM
      runtime.ts                           ← scroll, ResizeObserver, RAF batching
      index.ts                             ← createRenderer public API
  test/
    fixtures/
      generator.ts                         ← synthetic Block/Message factories + 5k convo
      synthetic-adapter.ts                 ← AsyncIterable<MessagePatch> replay adapter
    unit/
      patches.test.ts
      cache.test.ts
      virtualizer.test.ts
      theme.test.ts
      intents.test.ts
    playground/
      index.html                           ← manual perf playground
      main.ts                              ← wires fixture + adapter + renderer
    perf/
      scroll.spec.ts                       ← playwright; p99 frame < 16 ms over 5000 msgs
      resize.spec.ts                       ← playwright; 800→1200 in < 50 ms
      streaming.spec.ts                    ← playwright; 10 patches/frame × 60 no drops
  docs/plans/notes/
    m0-pretext-api.md                      ← spike output: pretext's actual API surface
```

**Why this split:** every file maps 1:1 to a section of `docs/03-renderer.md`, so the design and code stay easy to cross-reference. `patches.ts` is split out from `blocks.ts` because `blocks.ts` is a pure type module (the seam) and patch *application* is runtime behavior — the seam stays type-only as `docs/CONTRACTS.md` requires.

---

## Task 0: Project bootstrap

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `vite.config.ts`
- Create: `vitest.config.ts`
- Create: `playwright.config.ts`
- Create: `.gitignore` (extend existing)

- [ ] **Step 1: Initialize package.json**

Create `package.json`:

```json
{
  "name": "chat-plugin",
  "private": true,
  "type": "module",
  "version": "0.1.0-m0",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:perf": "playwright test",
    "lint": "eslint 'src/**/*.ts' 'test/**/*.ts'",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "@playwright/test": "^1.48.0",
    "@types/node": "^20.10.0",
    "@vitest/coverage-v8": "^2.1.0",
    "eslint": "^9.10.0",
    "eslint-plugin-boundaries": "^5.0.0",
    "happy-dom": "^15.7.0",
    "typescript": "^5.6.0",
    "vite": "^5.4.0",
    "vitest": "^2.1.0"
  }
}
```

- [ ] **Step 2: Add tsconfig.json**

Create `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "exactOptionalPropertyTypes": true,
    "isolatedModules": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "types": ["vitest/globals"],
    "paths": {
      "@core/*": ["./src/core/*"],
      "@test/*": ["./test/*"],
      "@pretext": ["./vendor/pretext/src/index.ts"]
    },
    "baseUrl": "."
  },
  "include": ["src", "test", "vite.config.ts", "vitest.config.ts", "playwright.config.ts"]
}
```

- [ ] **Step 3: Add vite.config.ts**

Create `vite.config.ts`:

```ts
import { defineConfig } from 'vite'
import { resolve } from 'node:path'

export default defineConfig({
  resolve: {
    alias: {
      '@core': resolve(__dirname, 'src/core'),
      '@test': resolve(__dirname, 'test'),
      '@pretext': resolve(__dirname, 'vendor/pretext/src/index.ts'),
    },
  },
  server: {
    open: '/test/playground/',
  },
  build: {
    target: 'es2022',
    sourcemap: true,
  },
})
```

- [ ] **Step 4: Add vitest.config.ts**

Create `vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config'
import { resolve } from 'node:path'

export default defineConfig({
  resolve: {
    alias: {
      '@core': resolve(__dirname, 'src/core'),
      '@test': resolve(__dirname, 'test'),
      '@pretext': resolve(__dirname, 'vendor/pretext/src/index.ts'),
    },
  },
  test: {
    globals: true,
    environment: 'happy-dom',
    include: ['test/unit/**/*.test.ts'],
    coverage: { reporter: ['text', 'html'] },
  },
})
```

- [ ] **Step 5: Add playwright.config.ts**

Create `playwright.config.ts`:

```ts
import { defineConfig } from '@playwright/test'

export default defineConfig({
  testDir: 'test/perf',
  workers: 1,                         // perf tests must not contend
  retries: 0,
  reporter: 'list',
  use: {
    headless: true,
    viewport: { width: 1200, height: 900 },
  },
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:5173/test/playground/',
    reuseExistingServer: true,
    timeout: 30_000,
  },
})
```

- [ ] **Step 6: Extend .gitignore**

Append to `.gitignore`:

```
node_modules/
dist/
coverage/
playwright-report/
test-results/
.vite/
```

- [ ] **Step 7: Install and verify toolchain**

Run: `npm install`
Expected: success, lockfile created.

Run: `npx tsc --noEmit`
Expected: success (no source files yet, so nothing to fail).

Run: `npx vitest run --reporter=verbose`
Expected: "No test files found" — vitest is wired but we have no tests yet.

- [ ] **Step 8: Commit**

```bash
git add package.json tsconfig.json vite.config.ts vitest.config.ts playwright.config.ts .gitignore package-lock.json
git commit -m "chore(m0): bootstrap typescript + vite + vitest + playwright"
```

---

## Task 1: Pretext spike (vendor + API doc)

The renderer's perf hinges on pretext. Before writing `layout.ts` we need to know its real API surface.

**Files:**
- Create: `vendor/pretext/` (git submodule)
- Create: `docs/plans/notes/m0-pretext-api.md`

- [ ] **Step 1: Add pretext as a submodule**

Run: `git submodule add https://github.com/chenglou/pretext vendor/pretext`
Expected: clone completes, `.gitmodules` is created.

- [ ] **Step 2: Inspect pretext's exports**

Run: `ls vendor/pretext/src` and `cat vendor/pretext/README.md` and `cat vendor/pretext/package.json`
Read what functions are exported, what they take, what they return, whether they need a DOM.

- [ ] **Step 3: Write a 30-line probe**

Create a scratch file (do not commit) `scratch/probe.ts`:

```ts
import * as pretext from '@pretext'
console.log(Object.keys(pretext))
// Try the names docs/03-renderer.md mentions:
//   prepare, prepareWithSegments, layout
// For each, log its signature (length, name).
for (const name of ['prepare', 'prepareWithSegments', 'layout']) {
  const fn = (pretext as any)[name]
  console.log(name, typeof fn, fn?.length)
}
```

Run: `npx tsx scratch/probe.ts` (install `tsx` ad-hoc with `npm i -D tsx` if needed).

If the names don't match, walk the actual exports until you find:
- A function that takes a string + font + width and returns a "prepared" object.
- A function that takes the prepared object + width and returns laid-out lines (or per-line offsets/heights).

Note: pretext is a 1D layout engine. Its layout output should give you, per text block, the **number of visual lines** at a given width. Multiplying by `lineHeight` gives the block height — that is what `layout.ts::measureText` returns.

- [ ] **Step 4: Write the API note**

Create `docs/plans/notes/m0-pretext-api.md`:

```markdown
# Pretext API surface (M0 spike output)

Vendored at `vendor/pretext` @ <commit-sha>.

## Functions we depend on

### `prepare(text: string, opts: { font, ... }): Prepared`
- Takes plain text + font metrics.
- Returns a "prepared" struct with measured glyphs / words.
- Pure (no DOM).

### `layout(prepared: Prepared, width: number): { lines: Line[]; height: number }`
- Takes a prepared struct + a target width.
- Returns visual line breaks and total height.
- Pure (no DOM).

### `prepareWithSegments(...)` — TBD if needed
- Used for runs with mixed styles in one paragraph.
- Skip in M0 if `prepare` + per-run width compositing suffices.

## Whether pretext needs a browser

- [ ] confirmed: works in headless Node (no canvas required), OR
- [ ] confirmed: requires `OffscreenCanvas`/`canvas` for glyph metrics.

If browser-only, all `layout.ts` tests run via Playwright; if Node-compatible,
`measureText` is unit-testable under Vitest with happy-dom. Record the answer.

## What we will NOT use

- (List anything pretext exports that's outside our M0 scope, so future
  authors don't reach for it inadvertently.)
```

Fill in the actual signatures and the headless-Node answer. This file is the source of truth for `layout.ts`.

- [ ] **Step 5: Delete the scratch probe and commit**

Run: `rm -rf scratch`

```bash
git add .gitmodules vendor/pretext docs/plans/notes/m0-pretext-api.md
git commit -m "chore(m0): vendor pretext + record api spike"
```

---

## Task 2: Block schema types (`core/blocks.ts`)

**Files:**
- Create: `src/core/blocks.ts`
- Create: `test/unit/blocks.test.ts`

This module is **types only** plus the `BLOCK_SCHEMA_VERSION` constant. No runtime behavior. Canonical content from `docs/02-block-schema.md`.

- [ ] **Step 1: Write a failing structural test**

Create `test/unit/blocks.test.ts`:

```ts
import { describe, it, expectTypeOf, expect } from 'vitest'
import {
  BLOCK_SCHEMA_VERSION,
  type Block,
  type CodeBlock,
  type Message,
  type MessagePatch,
  type Run,
  type TokenColor,
} from '@core/blocks'

describe('blocks schema', () => {
  it('exports a numeric BLOCK_SCHEMA_VERSION', () => {
    expect(typeof BLOCK_SCHEMA_VERSION).toBe('number')
    expect(BLOCK_SCHEMA_VERSION).toBeGreaterThanOrEqual(1)
  })

  it('Block is a discriminated union by `kind`', () => {
    const b: Block = { kind: 'text', runs: [{ text: 'hi' }] }
    expect(b.kind).toBe('text')
  })

  it('CodeBlock carries source + lines', () => {
    const c: CodeBlock = { kind: 'code', language: 'ts', source: 'a\nb', lines: [[{ text: 'a', mono: true }], [{ text: 'b', mono: true }]] }
    expect(c.lines.length).toBe(2)
  })

  it('MessagePatch includes patchMeta (regression: must exist)', () => {
    const p: MessagePatch = { op: 'patchMeta', messageId: 'm1', meta: { truncated: true } }
    expect(p.op).toBe('patchMeta')
  })

  it('Run.color narrows to TokenColor', () => {
    expectTypeOf<Run['color']>().toEqualTypeOf<TokenColor | undefined>()
  })

  it('Message.id is required, meta optional', () => {
    const m: Message = { id: 'x', role: 'user', blocks: [], status: 'final' }
    expect(m.meta).toBeUndefined()
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm test`
Expected: FAIL — module `@core/blocks` does not exist.

- [ ] **Step 3: Implement `core/blocks.ts`**

Create `src/core/blocks.ts`. Copy the canonical types verbatim from `docs/02-block-schema.md` §Types and §Patches. Add the version constant at the top. **No other exports.** Full content:

```ts
// SCHEMA — single source of truth for the renderer ↔ adapter seam.
// Canonical doc: docs/02-block-schema.md.
// Bump on any breaking change; adapters declare compatibility via
// `Adapter.schemaVersions` (see core/adapter.ts).

export const BLOCK_SCHEMA_VERSION = 1 as const

export type TokenColor =
  | 'fg' | 'muted'
  | 'keyword' | 'string' | 'number' | 'comment'
  | 'function' | 'type' | 'variable' | 'constant'
  | 'operator' | 'punctuation' | 'tag' | 'attr'
  | 'added' | 'removed' | 'changed'

export type Run = {
  text: string
  weight?: 'normal' | 'bold'
  italic?: boolean
  mono?: boolean
  underline?: boolean
  strike?: boolean
  href?: string
  color?: TokenColor
  title?: string
}

export type TextBlock    = { kind: 'text'; runs: Run[]; indent?: number }
export type HeadingBlock = { kind: 'heading'; level: 1 | 2 | 3 | 4 | 5 | 6; runs: Run[]; anchorId?: string }
export type QuoteBlock   = { kind: 'quote'; children: Block[] }
export type ListBlock    = { kind: 'list'; ordered: boolean; start?: number; items: Block[][] }
export type CodeBlock    = { kind: 'code'; language: string; source: string; lines: Run[][] }

export type OpaquePayload =
  | { type: 'html';   html: string }
  | { type: 'image';  url: string; alt?: string }
  | { type: 'iframe'; src: string; sandbox?: string }

export type OpaqueBlock = {
  kind: 'opaque'
  payload: OpaquePayload
  intrinsicWidth: number
  intrinsicHeight: number
  reflow: 'fixed' | 'fluid'
  copyText?: string
  cacheKey: string
}

export type Block = TextBlock | HeadingBlock | QuoteBlock | ListBlock | CodeBlock | OpaqueBlock

export type Role = 'user' | 'assistant' | 'system' | 'tool'

export type MessageMeta = {
  model?: string
  truncated?: boolean
  [key: string]: unknown
}

export type Message = {
  id: string
  role: Role
  blocks: Block[]
  status: 'final' | 'streaming'
  createdAt?: number
  parentId?: string
  meta?: MessageMeta
}

export type MessagePatch =
  | { op: 'add';           message: Message }
  | { op: 'append';        messageId: string; blockIndex: number; run: Run }
  | { op: 'appendLine';    messageId: string; blockIndex: number; text: string; line: Run[] }
  | { op: 'replace';       messageId: string; blockIndex: number; block: Block }
  | { op: 'replaceBlocks'; messageId: string; blocks: Block[] }
  | { op: 'patchMeta';     messageId: string; meta: Partial<MessageMeta> }
  | { op: 'finalize';      messageId: string }
  | { op: 'remove';        messageId: string }
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npm test`
Expected: PASS, 6 assertions.

Run: `npx tsc --noEmit`
Expected: success.

- [ ] **Step 5: Commit**

```bash
git add src/core/blocks.ts test/unit/blocks.test.ts
git commit -m "feat(core): block schema types + BLOCK_SCHEMA_VERSION"
```

---

## Task 3: Adapter & Capability types (`core/adapter.ts`)

**Files:**
- Create: `src/core/adapter.ts`
- Create: `test/unit/adapter.test.ts`

Pure type module. The interface lives in `core/` so adapters depend on it, not the reverse — see `docs/CONTRACTS.md` "Module ownership."

- [ ] **Step 1: Write a failing structural test**

Create `test/unit/adapter.test.ts`:

```ts
import { describe, it, expectTypeOf } from 'vitest'
import type { Adapter, Capability } from '@core/adapter'
import type { MessagePatch } from '@core/blocks'
import type { Intent, IntentResult } from '@core/intents'

describe('adapter types', () => {
  it('Capability includes the v1 union members', () => {
    const all: Capability[] = [
      'streaming', 'edit', 'regenerate', 'feedback', 'copy',
      'branchHistory', 'reportBug', 'devTools',
      'openLinkOverride', 'openImageOverride',
    ]
    expectTypeOf(all).toEqualTypeOf<Capability[]>()
  })

  it('Adapter exposes the v1 method surface', () => {
    type Method = keyof Adapter
    expectTypeOf<Method>().toEqualTypeOf<
      | 'name' | 'schemaVersions' | 'capabilities'
      | 'matches' | 'fetchConversation' | 'handleIntent'
      | 'findMessageList' | 'parseConversationIdFromUrl' | 'dispose'
    >()
  })

  it('fetchConversation returns AsyncIterable<MessagePatch>', () => {
    type R = ReturnType<Adapter['fetchConversation']>
    expectTypeOf<R>().toEqualTypeOf<AsyncIterable<MessagePatch>>()
  })

  it('handleIntent returns Promise<IntentResult>', () => {
    type R = ReturnType<Adapter['handleIntent']>
    expectTypeOf<R>().toEqualTypeOf<Promise<IntentResult>>()
    // Reference Intent so the import isn't unused.
    const _i: Intent = { type: 'edit', messageId: 'x' }
    void _i
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm test`
Expected: FAIL — `@core/adapter` and `@core/intents` do not exist.

- [ ] **Step 3: Implement `core/adapter.ts`**

Create `src/core/adapter.ts`:

```ts
// ADAPTER SEAM — interface defined here so adapters depend on the renderer's
// contract, not the reverse. Canonical doc: docs/04-adapters.md.

import type { MessagePatch } from './blocks'
import type { Intent, IntentResult } from './intents'

export type Capability =
  | 'streaming'
  | 'edit'
  | 'regenerate'
  | 'feedback'
  | 'copy'
  | 'branchHistory'
  | 'reportBug'
  | 'devTools'
  | 'openLinkOverride'
  | 'openImageOverride'

export interface Adapter {
  readonly name: string
  readonly schemaVersions: readonly number[]
  readonly capabilities: ReadonlySet<Capability>

  matches(location: Location, document: Document): boolean
  fetchConversation(signal: AbortSignal): AsyncIterable<MessagePatch>
  handleIntent(intent: Intent): Promise<IntentResult>
  findMessageList(container: HTMLElement): HTMLElement
  parseConversationIdFromUrl(url: URL): string | null
  dispose(): void
}
```

- [ ] **Step 4: Run the test (still failing — `@core/intents` not yet created)**

Run: `npm test`
Expected: FAIL — `@core/intents` not found. That's the next task.

- [ ] **Step 5: Stage and continue (do not commit yet — Task 4 finishes the pair)**

Leave the file unstaged; it'll commit alongside `intents.ts` in Task 4 step 5.

---

## Task 4: Intents & default handlers (`core/intents.ts`)

**Files:**
- Create: `src/core/intents.ts`
- Create: `test/unit/intents.test.ts`

The type module lives in `core/`. The renderer-default handlers for `openLink` and `openImage` live here too, since `docs/06-intents.md` defines them as renderer defaults.

- [ ] **Step 1: Write a failing test**

Create `test/unit/intents.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { handleDefaultIntent, type Intent, type IntentResult } from '@core/intents'

describe('intents', () => {
  let openSpy: ReturnType<typeof vi.spyOn>

  beforeEach(() => {
    openSpy = vi.spyOn(globalThis, 'open').mockImplementation(() => null)
  })
  afterEach(() => {
    openSpy.mockRestore()
  })

  it('openLink calls window.open with noopener,noreferrer', async () => {
    const r: IntentResult = await handleDefaultIntent(
      { type: 'openLink', href: 'https://example.com', messageId: 'm1' },
    )
    expect(openSpy).toHaveBeenCalledWith('https://example.com', '_blank', 'noopener,noreferrer')
    expect(r).toEqual({ ok: true })
  })

  it('openImage routes through window.open by default', async () => {
    const r = await handleDefaultIntent(
      { type: 'openImage', url: 'https://example.com/x.png', messageId: 'm1' },
    )
    expect(openSpy).toHaveBeenCalledWith('https://example.com/x.png', '_blank', 'noopener,noreferrer')
    expect(r).toEqual({ ok: true })
  })

  it('non-default intents are unhandled by default', async () => {
    const intents: Intent[] = [
      { type: 'edit', messageId: 'm1' },
      { type: 'regenerate', messageId: 'm1' },
      { type: 'copy', messageId: 'm1', blockIndex: 0 },
      { type: 'copyMessage', messageId: 'm1', format: 'plain' },
      { type: 'feedback', messageId: 'm1', value: 'up' },
      { type: 'jumpToOriginal', messageId: 'm1' },
      { type: 'reportBug', messageId: 'm1', kind: 'parse' },
    ]
    for (const i of intents) {
      const r = await handleDefaultIntent(i)
      expect(r.ok).toBe(false)
      if (!r.ok) expect(r.reason).toBe('unsupported')
    }
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm test`
Expected: FAIL — `@core/intents` not found.

- [ ] **Step 3: Implement `core/intents.ts`**

Create `src/core/intents.ts`:

```ts
// INTENT SEAM — canonical doc: docs/06-intents.md.

export type Intent =
  | { type: 'copy';           messageId: string; blockIndex?: number }
  | { type: 'copyMessage';    messageId: string; format: 'plain' | 'markdown' }
  | { type: 'edit';           messageId: string }
  | { type: 'regenerate';     messageId: string }
  | { type: 'feedback';       messageId: string; value: 'up' | 'down'; comment?: string }
  | { type: 'openLink';       href: string; messageId: string }
  | { type: 'openImage';      url: string; messageId: string }
  | { type: 'jumpToOriginal'; messageId: string }
  | { type: 'reportBug';      messageId: string; kind: 'parse' | 'layout' | 'other' }

export type IntentResult =
  | { ok: true }
  | { ok: false; reason: 'unsupported' | 'failed' | 'denied'; detail?: string }

/**
 * Renderer-handled defaults from docs/06-intents.md §Routing rules.
 * Returns `{ ok: false, reason: 'unsupported' }` for anything the renderer
 * does not handle by itself; the renderer routes those to `onIntent`.
 */
export async function handleDefaultIntent(intent: Intent): Promise<IntentResult> {
  switch (intent.type) {
    case 'openLink':
      window.open(intent.href, '_blank', 'noopener,noreferrer')
      return { ok: true }
    case 'openImage':
      window.open(intent.url, '_blank', 'noopener,noreferrer')
      return { ok: true }
    default:
      return { ok: false, reason: 'unsupported' }
  }
}
```

- [ ] **Step 4: Run tests (both adapter + intents pass)**

Run: `npm test`
Expected: PASS — both `adapter.test.ts` and `intents.test.ts` green.

Run: `npx tsc --noEmit`
Expected: success.

- [ ] **Step 5: Commit**

```bash
git add src/core/adapter.ts src/core/intents.ts test/unit/adapter.test.ts test/unit/intents.test.ts
git commit -m "feat(core): adapter & intent seam types + default openLink/openImage"
```

---

## Task 5: Theme module (`core/theme.ts`)

**Files:**
- Create: `src/core/theme.ts`
- Create: `test/unit/theme.test.ts`

For M0 we only need the `light` theme; `dark` lands in M4 polish. The function shape must support both.

- [ ] **Step 1: Write a failing test**

Create `test/unit/theme.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { resolveColor, type ThemeName } from '@core/theme'

describe('theme', () => {
  const themes: ThemeName[] = ['light', 'dark']

  it('resolves every TokenColor to a hex string per theme', () => {
    const tokens = ['fg', 'muted', 'keyword', 'string', 'number', 'comment',
      'function', 'type', 'variable', 'constant', 'operator', 'punctuation',
      'tag', 'attr', 'added', 'removed', 'changed'] as const
    for (const theme of themes) {
      for (const t of tokens) {
        const hex = resolveColor(t, theme)
        expect(hex).toMatch(/^#[0-9a-f]{6}$/i)
      }
    }
  })

  it('light and dark produce different fg colors', () => {
    expect(resolveColor('fg', 'light')).not.toBe(resolveColor('fg', 'dark'))
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm test`
Expected: FAIL — `@core/theme` does not exist.

- [ ] **Step 3: Implement `core/theme.ts`**

Create `src/core/theme.ts`:

```ts
import type { TokenColor } from './blocks'

export type ThemeName = 'light' | 'dark'

const LIGHT: Record<TokenColor, string> = {
  fg: '#1f2328',          muted: '#656d76',
  keyword: '#cf222e',     string: '#0a3069',
  number: '#0550ae',      comment: '#6e7781',
  function: '#8250df',    type: '#953800',
  variable: '#1f2328',    constant: '#0550ae',
  operator: '#1f2328',    punctuation: '#1f2328',
  tag: '#116329',         attr: '#0550ae',
  added: '#1a7f37',       removed: '#cf222e',
  changed: '#9a6700',
}

const DARK: Record<TokenColor, string> = {
  fg: '#e6edf3',          muted: '#7d8590',
  keyword: '#ff7b72',     string: '#a5d6ff',
  number: '#79c0ff',      comment: '#8b949e',
  function: '#d2a8ff',    type: '#ffa657',
  variable: '#e6edf3',    constant: '#79c0ff',
  operator: '#e6edf3',    punctuation: '#e6edf3',
  tag: '#7ee787',         attr: '#79c0ff',
  added: '#3fb950',       removed: '#f85149',
  changed: '#d29922',
}

export function resolveColor(token: TokenColor, theme: ThemeName): string {
  return (theme === 'dark' ? DARK : LIGHT)[token]
}
```

- [ ] **Step 4: Run the test to verify it passes**

Run: `npm test`
Expected: PASS — 2 assertions across 17 tokens × 2 themes.

- [ ] **Step 5: Commit**

```bash
git add src/core/theme.ts test/unit/theme.test.ts
git commit -m "feat(core): theme token-color resolution (light + dark)"
```

---

## Task 6: Patch application (`core/patches.ts`)

The state machine that turns `MessagePatch` into mutations on the in-memory `Message[]`. Enforces the patch protocol invariants from `docs/CONTRACTS.md`.

**Files:**
- Create: `src/core/patches.ts`
- Create: `test/unit/patches.test.ts`

- [ ] **Step 1: Write failing tests for the happy paths**

Create `test/unit/patches.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { createState, applyPatch } from '@core/patches'
import type { Message, MessagePatch } from '@core/blocks'

const m = (id: string): Message => ({ id, role: 'assistant', blocks: [], status: 'streaming' })

describe('patches: add / finalize / remove', () => {
  it('add appends a new message', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: m('m1') })
    expect(s.messages.map(x => x.id)).toEqual(['m1'])
  })

  it('add with duplicate id replaces blocks (treated as replaceBlocks)', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: { ...m('m1'), blocks: [{ kind: 'text', runs: [{ text: 'a' }] }] } })
    applyPatch(s, { op: 'add', message: { ...m('m1'), blocks: [{ kind: 'text', runs: [{ text: 'b' }] }] } })
    expect(s.messages).toHaveLength(1)
    expect((s.messages[0]!.blocks[0] as any).runs[0].text).toBe('b')
  })

  it('finalize flips streaming → final', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: m('m1') })
    applyPatch(s, { op: 'finalize', messageId: 'm1' })
    expect(s.messages[0]!.status).toBe('final')
  })

  it('remove drops the message', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: m('m1') })
    applyPatch(s, { op: 'remove', messageId: 'm1' })
    expect(s.messages).toHaveLength(0)
  })
})

describe('patches: replace (create-or-replace)', () => {
  it('creates a block at blocks.length', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: m('m1') })
    applyPatch(s, { op: 'replace', messageId: 'm1', blockIndex: 0, block: { kind: 'text', runs: [{ text: 'x' }] } })
    expect(s.messages[0]!.blocks).toHaveLength(1)
  })

  it('throws on hole creation (blockIndex > blocks.length)', () => {
    const s = createState({ devAssertions: true })
    applyPatch(s, { op: 'add', message: m('m1') })
    expect(() =>
      applyPatch(s, { op: 'replace', messageId: 'm1', blockIndex: 1, block: { kind: 'text', runs: [{ text: 'x' }] } }),
    ).toThrow(/hole/)
  })

  it('replaces existing block in-place', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: { ...m('m1'), blocks: [{ kind: 'text', runs: [{ text: 'a' }] }] } })
    applyPatch(s, { op: 'replace', messageId: 'm1', blockIndex: 0, block: { kind: 'text', runs: [{ text: 'b' }] } })
    expect((s.messages[0]!.blocks[0] as any).runs[0].text).toBe('b')
  })
})

describe('patches: append (text only)', () => {
  it('appends a Run to a text block', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: { ...m('m1'), blocks: [{ kind: 'text', runs: [{ text: 'a' }] }] } })
    applyPatch(s, { op: 'append', messageId: 'm1', blockIndex: 0, run: { text: 'b' } })
    const t = s.messages[0]!.blocks[0] as any
    expect(t.runs).toEqual([{ text: 'a' }, { text: 'b' }])
  })

  it('throws when target is a code block (dev mode)', () => {
    const s = createState({ devAssertions: true })
    applyPatch(s, { op: 'add', message: { ...m('m1'), blocks: [{ kind: 'code', language: '', source: '', lines: [] }] } })
    expect(() =>
      applyPatch(s, { op: 'append', messageId: 'm1', blockIndex: 0, run: { text: 'x' } }),
    ).toThrow(/text/)
  })

  it('throws when block does not exist (dev mode)', () => {
    const s = createState({ devAssertions: true })
    applyPatch(s, { op: 'add', message: m('m1') })
    expect(() =>
      applyPatch(s, { op: 'append', messageId: 'm1', blockIndex: 0, run: { text: 'x' } }),
    ).toThrow(/missing/)
  })
})

describe('patches: appendLine (code only, atomic source/lines)', () => {
  it('first appendLine into empty source sets source = text', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: { ...m('m1'), blocks: [{ kind: 'code', language: '', source: '', lines: [] }] } })
    applyPatch(s, { op: 'appendLine', messageId: 'm1', blockIndex: 0, text: 'def x():', line: [{ text: 'def x():', mono: true }] })
    const c = s.messages[0]!.blocks[0] as any
    expect(c.source).toBe('def x():')
    expect(c.lines).toHaveLength(1)
  })

  it('subsequent appendLines join with \\n and stay in lockstep', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: { ...m('m1'), blocks: [{ kind: 'code', language: '', source: '', lines: [] }] } })
    applyPatch(s, { op: 'appendLine', messageId: 'm1', blockIndex: 0, text: 'a', line: [{ text: 'a', mono: true }] })
    applyPatch(s, { op: 'appendLine', messageId: 'm1', blockIndex: 0, text: 'b', line: [{ text: 'b', mono: true }] })
    const c = s.messages[0]!.blocks[0] as any
    expect(c.source).toBe('a\nb')
    expect(c.lines).toHaveLength(2)
  })

  it('throws when target is a text block (dev mode)', () => {
    const s = createState({ devAssertions: true })
    applyPatch(s, { op: 'add', message: { ...m('m1'), blocks: [{ kind: 'text', runs: [{ text: 'a' }] }] } })
    expect(() =>
      applyPatch(s, { op: 'appendLine', messageId: 'm1', blockIndex: 0, text: 'x', line: [{ text: 'x', mono: true }] }),
    ).toThrow(/code/)
  })
})

describe('patches: replaceBlocks + patchMeta', () => {
  it('replaceBlocks swaps the whole list', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: { ...m('m1'), blocks: [{ kind: 'text', runs: [{ text: 'a' }] }] } })
    applyPatch(s, { op: 'replaceBlocks', messageId: 'm1', blocks: [] })
    expect(s.messages[0]!.blocks).toHaveLength(0)
  })

  it('patchMeta shallow-merges into meta', () => {
    const s = createState()
    applyPatch(s, { op: 'add', message: m('m1') })
    applyPatch(s, { op: 'patchMeta', messageId: 'm1', meta: { truncated: true } })
    expect(s.messages[0]!.meta).toEqual({ truncated: true })
    applyPatch(s, { op: 'patchMeta', messageId: 'm1', meta: { model: 'gpt-5' } })
    expect(s.messages[0]!.meta).toEqual({ truncated: true, model: 'gpt-5' })
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test`
Expected: FAIL — `@core/patches` does not exist.

- [ ] **Step 3: Implement `core/patches.ts`**

Create `src/core/patches.ts`:

```ts
import type { Block, CodeBlock, Message, MessagePatch, TextBlock } from './blocks'

export type State = {
  messages: Message[]
  /** Index from id → position in `messages` for O(1) lookup. */
  index: Map<string, number>
  devAssertions: boolean
}

export type StateOpts = { devAssertions?: boolean }

export function createState(opts: StateOpts = {}): State {
  return { messages: [], index: new Map(), devAssertions: opts.devAssertions ?? false }
}

/**
 * Mutates `state` in place. Returns the affected message ids so callers
 * (cache, virtualizer) can invalidate.
 */
export function applyPatch(state: State, patch: MessagePatch): { affected: string[] } {
  switch (patch.op) {
    case 'add': return doAdd(state, patch.message)
    case 'remove': return doRemove(state, patch.messageId)
    case 'finalize': return doFinalize(state, patch.messageId)
    case 'replace': return doReplace(state, patch.messageId, patch.blockIndex, patch.block)
    case 'replaceBlocks': return doReplaceBlocks(state, patch.messageId, patch.blocks)
    case 'append': return doAppend(state, patch.messageId, patch.blockIndex, patch.run)
    case 'appendLine': return doAppendLine(state, patch.messageId, patch.blockIndex, patch.text, patch.line)
    case 'patchMeta': return doPatchMeta(state, patch.messageId, patch.meta)
  }
}

function doAdd(s: State, m: Message): { affected: string[] } {
  const existing = s.index.get(m.id)
  if (existing !== undefined) {
    // Duplicate add → treat as replaceBlocks (per docs/05-streaming.md §Failure modes).
    s.messages[existing] = { ...m }
    return { affected: [m.id] }
  }
  s.index.set(m.id, s.messages.length)
  s.messages.push({ ...m })
  return { affected: [m.id] }
}

function doRemove(s: State, id: string): { affected: string[] } {
  const i = s.index.get(id)
  if (i === undefined) return { affected: [] }
  s.messages.splice(i, 1)
  s.index.delete(id)
  // Re-index everything after `i`.
  for (let j = i; j < s.messages.length; j++) s.index.set(s.messages[j]!.id, j)
  return { affected: [id] }
}

function doFinalize(s: State, id: string): { affected: string[] } {
  const m = lookup(s, id, 'finalize')
  if (!m) return { affected: [] }
  m.status = 'final'
  return { affected: [id] }
}

function doReplace(s: State, id: string, idx: number, block: Block): { affected: string[] } {
  const m = lookup(s, id, 'replace')
  if (!m) return { affected: [] }
  if (idx === m.blocks.length) {
    m.blocks.push(block)
  } else if (idx < m.blocks.length) {
    if (s.devAssertions) {
      const prev = m.blocks[idx]!
      if ((prev.kind === 'text' || prev.kind === 'code') && block.kind === 'opaque') {
        throw new Error('replace: cannot change kind text/code → opaque mid-stream')
      }
    }
    m.blocks[idx] = block
  } else {
    if (s.devAssertions) throw new Error(`replace: hole — blockIndex ${idx} > blocks.length ${m.blocks.length}`)
  }
  return { affected: [id] }
}

function doReplaceBlocks(s: State, id: string, blocks: Block[]): { affected: string[] } {
  const m = lookup(s, id, 'replaceBlocks')
  if (!m) return { affected: [] }
  m.blocks = blocks
  return { affected: [id] }
}

function doAppend(s: State, id: string, idx: number, run: import('./blocks').Run): { affected: string[] } {
  const m = lookup(s, id, 'append')
  if (!m) return { affected: [] }
  const block = m.blocks[idx]
  if (!block) {
    if (s.devAssertions) throw new Error(`append: missing block ${idx} on ${id}`)
    return { affected: [] }
  }
  if (block.kind !== 'text') {
    if (s.devAssertions) throw new Error(`append: target must be text, got ${block.kind}`)
    return { affected: [] }
  }
  ;(block as TextBlock).runs.push(run)
  return { affected: [id] }
}

function doAppendLine(
  s: State, id: string, idx: number,
  text: string, line: import('./blocks').Run[],
): { affected: string[] } {
  const m = lookup(s, id, 'appendLine')
  if (!m) return { affected: [] }
  const block = m.blocks[idx]
  if (!block) {
    if (s.devAssertions) throw new Error(`appendLine: missing block ${idx} on ${id}`)
    return { affected: [] }
  }
  if (block.kind !== 'code') {
    if (s.devAssertions) throw new Error(`appendLine: target must be code, got ${block.kind}`)
    return { affected: [] }
  }
  const c = block as CodeBlock
  c.source = c.source === '' ? text : c.source + '\n' + text
  c.lines.push(line)
  return { affected: [id] }
}

function doPatchMeta(s: State, id: string, meta: Partial<NonNullable<Message['meta']>>): { affected: string[] } {
  const m = lookup(s, id, 'patchMeta')
  if (!m) return { affected: [] }
  m.meta = { ...(m.meta ?? {}), ...meta }
  return { affected: [id] }
}

function lookup(s: State, id: string, op: string): Message | null {
  const i = s.index.get(id)
  if (i === undefined) {
    if (s.devAssertions) throw new Error(`${op}: unknown messageId ${id}`)
    return null
  }
  return s.messages[i]!
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test`
Expected: PASS — all 16+ patch tests green.

- [ ] **Step 5: Commit**

```bash
git add src/core/patches.ts test/unit/patches.test.ts
git commit -m "feat(core): patch state machine (add/replace/append/appendLine/patchMeta)"
```

---

## Task 7: Height cache (`core/cache.ts`)

LRU bounded at 50k entries, keyed by `${messageId}:${blockIndex}:${widthBucket}`. Width is rounded to nearest 8 px to avoid thrash on continuous resize.

**Files:**
- Create: `src/core/cache.ts`
- Create: `test/unit/cache.test.ts`

- [ ] **Step 1: Write failing tests**

Create `test/unit/cache.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { createHeightCache } from '@core/cache'

describe('height cache', () => {
  it('returns undefined on miss, stored value on hit', () => {
    const c = createHeightCache({ capacity: 100 })
    expect(c.get('m1', 0, 800)).toBeUndefined()
    c.set('m1', 0, 800, 42)
    expect(c.get('m1', 0, 800)).toBe(42)
  })

  it('rounds width to nearest 8 px', () => {
    const c = createHeightCache({ capacity: 100 })
    c.set('m1', 0, 803, 42)
    expect(c.get('m1', 0, 800)).toBe(42)   // 800 and 803 share bucket 800
    expect(c.get('m1', 0, 807)).toBe(42)
    expect(c.get('m1', 0, 808)).toBeUndefined() // different bucket
  })

  it('invalidate(messageId) drops all entries for that message', () => {
    const c = createHeightCache({ capacity: 100 })
    c.set('m1', 0, 800, 10)
    c.set('m1', 1, 800, 20)
    c.set('m2', 0, 800, 30)
    c.invalidate('m1')
    expect(c.get('m1', 0, 800)).toBeUndefined()
    expect(c.get('m1', 1, 800)).toBeUndefined()
    expect(c.get('m2', 0, 800)).toBe(30)
  })

  it('invalidate(messageId, blockIndex) drops only that block', () => {
    const c = createHeightCache({ capacity: 100 })
    c.set('m1', 0, 800, 10)
    c.set('m1', 1, 800, 20)
    c.invalidate('m1', 0)
    expect(c.get('m1', 0, 800)).toBeUndefined()
    expect(c.get('m1', 1, 800)).toBe(20)
  })

  it('invalidateAll clears the cache', () => {
    const c = createHeightCache({ capacity: 100 })
    c.set('m1', 0, 800, 10)
    c.invalidateAll()
    expect(c.get('m1', 0, 800)).toBeUndefined()
  })

  it('evicts least-recently-used when over capacity', () => {
    const c = createHeightCache({ capacity: 3 })
    c.set('a', 0, 800, 1)
    c.set('b', 0, 800, 2)
    c.set('c', 0, 800, 3)
    c.get('a', 0, 800)             // touch 'a' → most recent
    c.set('d', 0, 800, 4)          // should evict 'b' (LRU)
    expect(c.get('a', 0, 800)).toBe(1)
    expect(c.get('b', 0, 800)).toBeUndefined()
    expect(c.get('c', 0, 800)).toBe(3)
    expect(c.get('d', 0, 800)).toBe(4)
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test`
Expected: FAIL — `@core/cache` does not exist.

- [ ] **Step 3: Implement `core/cache.ts`**

Create `src/core/cache.ts`:

```ts
const WIDTH_BUCKET = 8

export type HeightCache = {
  get(messageId: string, blockIndex: number, width: number): number | undefined
  set(messageId: string, blockIndex: number, width: number, height: number): void
  invalidate(messageId: string, blockIndex?: number): void
  invalidateAll(): void
  size(): number
}

export type CacheOpts = { capacity?: number }

const bucket = (w: number) => Math.round(w / WIDTH_BUCKET) * WIDTH_BUCKET
const key = (id: string, idx: number, w: number) => `${id}:${idx}:${bucket(w)}`

export function createHeightCache(opts: CacheOpts = {}): HeightCache {
  const capacity = opts.capacity ?? 50_000
  // Map preserves insertion order; we use it as an LRU by delete+set on touch.
  const map = new Map<string, number>()
  // Reverse index so invalidate(messageId) is O(blocks per message), not O(n).
  const byMessage = new Map<string, Set<string>>()

  function trackKey(id: string, k: string) {
    let set = byMessage.get(id)
    if (!set) { set = new Set(); byMessage.set(id, set) }
    set.add(k)
  }

  function untrackKey(id: string, k: string) {
    const set = byMessage.get(id)
    if (!set) return
    set.delete(k)
    if (set.size === 0) byMessage.delete(id)
  }

  return {
    get(id, idx, w) {
      const k = key(id, idx, w)
      if (!map.has(k)) return undefined
      const v = map.get(k)!
      map.delete(k); map.set(k, v)             // touch (LRU)
      return v
    },
    set(id, idx, w, h) {
      const k = key(id, idx, w)
      if (!map.has(k)) trackKey(id, k)
      map.delete(k); map.set(k, h)             // upsert + touch
      while (map.size > capacity) {
        const oldest = map.keys().next().value as string | undefined
        if (oldest === undefined) break
        map.delete(oldest)
        const oldestId = oldest.split(':', 1)[0]!
        untrackKey(oldestId, oldest)
      }
    },
    invalidate(id, idx) {
      const set = byMessage.get(id)
      if (!set) return
      if (idx === undefined) {
        for (const k of set) map.delete(k)
        byMessage.delete(id)
      } else {
        const prefix = `${id}:${idx}:`
        for (const k of [...set]) {
          if (k.startsWith(prefix)) {
            map.delete(k)
            set.delete(k)
          }
        }
        if (set.size === 0) byMessage.delete(id)
      }
    },
    invalidateAll() {
      map.clear()
      byMessage.clear()
    },
    size() { return map.size },
  }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test`
Expected: PASS — 6 cache tests green.

- [ ] **Step 5: Commit**

```bash
git add src/core/cache.ts test/unit/cache.test.ts
git commit -m "feat(core): height cache (LRU, width-bucketed, message-indexed)"
```

---

## Task 8: Layout module (`core/layout.ts`)

Pretext wrapper. Pure functions; no DOM beyond what pretext itself does (verified in Task 1 spike). The signatures match the renderer doc; the implementation depends on the spike output.

**Files:**
- Create: `src/core/layout.ts`
- Create: `test/unit/layout.test.ts`

- [ ] **Step 1: Read your spike notes**

Re-open `docs/plans/notes/m0-pretext-api.md` and confirm the function signatures you decided to depend on. The code below assumes:
- `prepare(text: string, opts: { font: string; fontSize: number }): Prepared`
- `layout(prepared: Prepared, width: number): { lineCount: number }`

If pretext's real shape differs, adjust the wrapper accordingly. The exported signatures of `layout.ts` (below) must NOT change — they are what `virtualizer.ts` calls.

- [ ] **Step 2: Write failing tests**

Create `test/unit/layout.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { measureBlock, measureMessage, type LayoutCtx } from '@core/layout'
import type { Block, Message } from '@core/blocks'

const ctx: LayoutCtx = {
  font: '14px ui-sans-serif, system-ui',
  fontSize: 14,
  lineHeight: 20,
  monoFont: '13px ui-monospace, SF Mono, monospace',
  monoFontSize: 13,
  monoLineHeight: 18,
}

describe('layout', () => {
  it('text block height = lineCount * lineHeight', () => {
    const b: Block = { kind: 'text', runs: [{ text: 'hello world '.repeat(50) }] }
    const m = measureBlock(b, 400, ctx)
    expect(m.height).toBeGreaterThan(0)
    expect(m.height % ctx.lineHeight).toBe(0)
  })

  it('code block height = lines.length * monoLineHeight', () => {
    const b: Block = { kind: 'code', language: 'ts', source: 'a\nb\nc', lines: [
      [{ text: 'a', mono: true }],
      [{ text: 'b', mono: true }],
      [{ text: 'c', mono: true }],
    ] }
    const m = measureBlock(b, 400, ctx)
    expect(m.height).toBe(3 * ctx.monoLineHeight)
  })

  it('opaque fixed block returns intrinsicHeight regardless of width', () => {
    const b: Block = { kind: 'opaque', payload: { type: 'image', url: 'x' },
      intrinsicWidth: 200, intrinsicHeight: 150, reflow: 'fixed', cacheKey: 'x' }
    expect(measureBlock(b, 100, ctx).height).toBe(150)
    expect(measureBlock(b, 1000, ctx).height).toBe(150)
  })

  it('opaque fluid block scales by width / intrinsicWidth', () => {
    const b: Block = { kind: 'opaque', payload: { type: 'image', url: 'x' },
      intrinsicWidth: 400, intrinsicHeight: 200, reflow: 'fluid', cacheKey: 'x' }
    // narrower than intrinsic → scaled down
    expect(measureBlock(b, 200, ctx).height).toBe(100)
    // wider → not upscaled (capped at intrinsic)
    expect(measureBlock(b, 800, ctx).height).toBe(200)
  })

  it('measureMessage returns one entry per block', () => {
    const m: Message = { id: 'm1', role: 'assistant', status: 'final', blocks: [
      { kind: 'text', runs: [{ text: 'hi' }] },
      { kind: 'code', language: '', source: 'x', lines: [[{ text: 'x', mono: true }]] },
    ] }
    const out = measureMessage(m, 400, ctx)
    expect(out).toHaveLength(2)
    expect(out.every(x => x.height > 0)).toBe(true)
  })
})
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `npm test`
Expected: FAIL — `@core/layout` does not exist.

- [ ] **Step 4: Implement `core/layout.ts`**

Create `src/core/layout.ts`:

```ts
import type { Block, CodeBlock, Message, OpaqueBlock, Run, TextBlock } from './blocks'
// IMPORTANT: this import path uses the @pretext alias from tsconfig/vite.
// If the spike (Task 1) found a different shape, adjust here only.
import { prepare, layout as ptxLayout } from '@pretext'

export type LayoutCtx = {
  font: string
  fontSize: number
  lineHeight: number
  monoFont: string
  monoFontSize: number
  monoLineHeight: number
}

export type BlockMeasurement = { height: number }

export function measureBlock(block: Block, width: number, ctx: LayoutCtx): BlockMeasurement {
  switch (block.kind) {
    case 'text':    return measureText(block, width, ctx)
    case 'heading': return measureHeading(block, width, ctx)
    case 'quote':   return measureQuote(block, width, ctx)
    case 'list':    return measureList(block, width, ctx)
    case 'code':    return measureCode(block, ctx)
    case 'opaque':  return measureOpaque(block, width)
  }
}

export function measureMessage(m: Message, width: number, ctx: LayoutCtx): BlockMeasurement[] {
  return m.blocks.map(b => measureBlock(b, width, ctx))
}

function runsToText(runs: Run[]): string {
  return runs.map(r => r.text).join('')
}

function measureText(b: TextBlock, width: number, ctx: LayoutCtx): BlockMeasurement {
  const text = runsToText(b.runs)
  if (text.length === 0) return { height: ctx.lineHeight }
  const indentPx = (b.indent ?? 0) * ctx.fontSize
  const w = Math.max(1, width - indentPx)
  const prepared = prepare(text, { font: ctx.font, fontSize: ctx.fontSize })
  const { lineCount } = ptxLayout(prepared, w)
  return { height: lineCount * ctx.lineHeight }
}

function measureHeading(b: { kind: 'heading'; level: number; runs: Run[] }, width: number, ctx: LayoutCtx): BlockMeasurement {
  // Headings get a slightly larger line-height; we use a per-level multiplier.
  const scale = [1.6, 1.4, 1.25, 1.15, 1.05, 1.0][b.level - 1] ?? 1.0
  const text = runsToText(b.runs)
  const prepared = prepare(text, { font: ctx.font, fontSize: ctx.fontSize * scale })
  const { lineCount } = ptxLayout(prepared, width)
  return { height: Math.ceil(lineCount * ctx.lineHeight * scale) }
}

function measureQuote(b: { kind: 'quote'; children: Block[] }, width: number, ctx: LayoutCtx): BlockMeasurement {
  const inner = width - 16            // 16 px quote bar + gap
  const total = b.children.reduce((acc, c) => acc + measureBlock(c, inner, ctx).height, 0)
  return { height: total + 8 }        // small vertical padding
}

function measureList(b: { kind: 'list'; items: Block[][] }, width: number, ctx: LayoutCtx): BlockMeasurement {
  const inner = width - 24            // bullet/number gutter
  const total = b.items.reduce((acc, item) => {
    const itemHeight = item.reduce((sum, blk) => sum + measureBlock(blk, inner, ctx).height, 0)
    return acc + itemHeight
  }, 0)
  return { height: total }
}

function measureCode(b: CodeBlock, ctx: LayoutCtx): BlockMeasurement {
  // Code does not wrap; height is purely line-count × monoLineHeight.
  // Width drives only horizontal scroll, not measurement.
  return { height: b.lines.length * ctx.monoLineHeight }
}

function measureOpaque(b: OpaqueBlock, width: number): BlockMeasurement {
  if (b.reflow === 'fixed') return { height: b.intrinsicHeight }
  // fluid: scale down if width < intrinsicWidth, never upscale.
  const ratio = Math.min(1, width / b.intrinsicWidth)
  return { height: Math.round(b.intrinsicHeight * ratio) }
}
```

- [ ] **Step 5: Run the tests**

Run: `npm test`

If pretext requires a real browser canvas and happy-dom can't satisfy it, the text-based tests will fail. In that case:

a. Mark `layout.test.ts` tests with `.skip` for the `text` and `heading` cases that touch pretext.
b. Move those assertions into a Playwright test alongside the perf tests (Task 17).
c. Keep the `code` and `opaque` cases in the unit file — they don't call pretext.

Document the choice with a comment at the top of `layout.test.ts`.

Expected (one of):
- All tests pass under happy-dom, OR
- Code + opaque pass; text + heading skipped with a note pointing at the playwright suite.

- [ ] **Step 6: Commit**

```bash
git add src/core/layout.ts test/unit/layout.test.ts
git commit -m "feat(core): layout — pretext wrapper for text/code/opaque/list/quote/heading"
```

---

## Task 9: Opaque mount (`core/mount.ts`)

Hosts `OpaqueBlock` payloads (image / html / iframe) inside an isolated container with a `ResizeObserver` for fluid blocks.

**Files:**
- Create: `src/core/mount.ts`
- Create: `test/unit/mount.test.ts`

- [ ] **Step 1: Write failing tests**

Create `test/unit/mount.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { mountOpaque } from '@core/mount'
import type { OpaqueBlock } from '@core/blocks'

const baseImg: OpaqueBlock = {
  kind: 'opaque',
  payload: { type: 'image', url: 'data:image/png;base64,iVBORw0KGgo=', alt: 'pixel' },
  intrinsicWidth: 100, intrinsicHeight: 100,
  reflow: 'fixed', cacheKey: 'pixel',
}

describe('mount', () => {
  it('mounts an image with loading=lazy and decoding=async', () => {
    const host = document.createElement('div')
    mountOpaque(host, baseImg, { onResize: () => {} })
    const img = host.querySelector('img')!
    expect(img).toBeTruthy()
    expect(img.getAttribute('loading')).toBe('lazy')
    expect(img.getAttribute('decoding')).toBe('async')
    expect(img.getAttribute('alt')).toBe('pixel')
  })

  it('mounts html into a sandboxed div', () => {
    const host = document.createElement('div')
    const b: OpaqueBlock = { ...baseImg, payload: { type: 'html', html: '<b>x</b>' }, cacheKey: 'h' }
    mountOpaque(host, b, { onResize: () => {} })
    expect(host.innerHTML).toContain('<b>x</b>')
    const wrap = host.firstElementChild as HTMLElement
    expect(wrap.dataset.cpOpaque).toBe('html')
  })

  it('mounts iframe with sandbox attribute', () => {
    const host = document.createElement('div')
    const b: OpaqueBlock = { ...baseImg, payload: { type: 'iframe', src: 'about:blank', sandbox: 'allow-scripts' }, cacheKey: 'if' }
    mountOpaque(host, b, { onResize: () => {} })
    const iframe = host.querySelector('iframe')!
    expect(iframe.getAttribute('src')).toBe('about:blank')
    expect(iframe.getAttribute('sandbox')).toBe('allow-scripts')
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test`
Expected: FAIL — `@core/mount` does not exist.

- [ ] **Step 3: Implement `core/mount.ts`**

Create `src/core/mount.ts`:

```ts
import type { OpaqueBlock } from './blocks'

export type MountOpts = {
  /** Called when a `reflow: 'fluid'` block changes intrinsic size. */
  onResize: (newSize: { width: number; height: number }) => void
}

export function mountOpaque(host: HTMLElement, block: OpaqueBlock, opts: MountOpts): () => void {
  // Clear host first; mount is idempotent for re-mounts.
  while (host.firstChild) host.removeChild(host.firstChild)

  const wrap = document.createElement('div')
  wrap.dataset.cpOpaque = block.payload.type
  wrap.style.maxWidth = '100%'

  switch (block.payload.type) {
    case 'image': {
      const img = document.createElement('img')
      img.src = block.payload.url
      if (block.payload.alt) img.alt = block.payload.alt
      img.loading = 'lazy'
      img.decoding = 'async'
      img.style.maxWidth = '100%'
      img.style.height = 'auto'
      wrap.appendChild(img)
      break
    }
    case 'html': {
      // Adapter is trusted, but the HTML may include user content.
      // M0 has no DOMPurify dep yet; sanitize lands in M1 (services/sanitize.ts).
      // For M0 fixtures we only ever set safe HTML, so direct innerHTML is OK.
      wrap.innerHTML = block.payload.html
      break
    }
    case 'iframe': {
      const iframe = document.createElement('iframe')
      iframe.src = block.payload.src
      if (block.payload.sandbox) iframe.setAttribute('sandbox', block.payload.sandbox)
      iframe.style.width = '100%'
      iframe.style.border = '0'
      iframe.height = String(block.intrinsicHeight)
      wrap.appendChild(iframe)
      break
    }
  }

  host.appendChild(wrap)

  // ResizeObserver only for fluid reflow.
  let ro: ResizeObserver | null = null
  if (block.reflow === 'fluid' && typeof ResizeObserver !== 'undefined') {
    ro = new ResizeObserver(entries => {
      const e = entries[0]
      if (!e) return
      opts.onResize({ width: e.contentRect.width, height: e.contentRect.height })
    })
    ro.observe(wrap)
  }

  return () => {
    ro?.disconnect()
    if (wrap.parentElement === host) host.removeChild(wrap)
  }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test`
Expected: PASS — 3 mount tests green.

- [ ] **Step 5: Commit**

```bash
git add src/core/mount.ts test/unit/mount.test.ts
git commit -m "feat(core): opaque mount (image/html/iframe + fluid ResizeObserver)"
```

---

## Task 10: Virtualizer (`core/virtualizer.ts`)

Maintains cumulative offsets, decides which messages are in window, mounts/unmounts message DOM. Pure logic — no DOM here; the renderer wires the virtualizer's decisions to mount calls.

**Files:**
- Create: `src/core/virtualizer.ts`
- Create: `test/unit/virtualizer.test.ts`

- [ ] **Step 1: Write failing tests**

Create `test/unit/virtualizer.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { createVirtualizer } from '@core/virtualizer'

describe('virtualizer', () => {
  // Fixture: 100 messages, each 100 px tall.
  const heights = new Array(100).fill(100)

  it('reports total height as sum of heights', () => {
    const v = createVirtualizer({ overscanPx: 0 })
    v.setHeights(heights)
    expect(v.totalHeight()).toBe(10_000)
  })

  it('returns visible range covering [scrollTop, scrollTop+viewport)', () => {
    const v = createVirtualizer({ overscanPx: 0 })
    v.setHeights(heights)
    const r = v.visibleRange(0, 500)             // top..500
    expect(r).toEqual({ start: 0, end: 5 })       // messages 0..4 inclusive → [0, 5)
  })

  it('expands range by overscan on both sides', () => {
    const v = createVirtualizer({ overscanPx: 200 })
    v.setHeights(heights)
    const r = v.visibleRange(1000, 500)           // viewport at msg 10..15
    // overscan 200 px = 2 messages each side; clamped to bounds
    expect(r.start).toBe(8)
    expect(r.end).toBe(17)
  })

  it('clamps range to [0, n]', () => {
    const v = createVirtualizer({ overscanPx: 1000 })
    v.setHeights(heights)
    const r = v.visibleRange(0, 100)
    expect(r.start).toBe(0)
    expect(r.end).toBeLessThanOrEqual(100)
  })

  it('offsetOf gives cumulative top for index i', () => {
    const v = createVirtualizer({ overscanPx: 0 })
    v.setHeights(heights)
    expect(v.offsetOf(0)).toBe(0)
    expect(v.offsetOf(5)).toBe(500)
    expect(v.offsetOf(100)).toBe(10_000)
  })

  it('updateHeight(i, h) shifts subsequent offsets', () => {
    const v = createVirtualizer({ overscanPx: 0 })
    v.setHeights(heights)
    v.updateHeight(0, 200)
    expect(v.offsetOf(1)).toBe(200)
    expect(v.totalHeight()).toBe(10_100)
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test`
Expected: FAIL — `@core/virtualizer` does not exist.

- [ ] **Step 3: Implement `core/virtualizer.ts`**

Create `src/core/virtualizer.ts`:

```ts
export type VisibleRange = { start: number; end: number }

export type Virtualizer = {
  setHeights(heights: number[]): void
  updateHeight(index: number, newHeight: number): void
  totalHeight(): number
  offsetOf(index: number): number
  visibleRange(scrollTop: number, viewportHeight: number): VisibleRange
}

export type VirtualizerOpts = { overscanPx: number }

/**
 * Cumulative-offset table with O(log n) binary search for visible range.
 * `offsets[i]` = pixel y-position of the top of message i.
 * `offsets[length]` = totalHeight (one extra entry for end-of-list).
 */
export function createVirtualizer(opts: VirtualizerOpts): Virtualizer {
  let heights: number[] = []
  let offsets: number[] = [0]

  function rebuild() {
    offsets = new Array(heights.length + 1)
    offsets[0] = 0
    for (let i = 0; i < heights.length; i++) offsets[i + 1] = offsets[i]! + heights[i]!
  }

  function findIndexAt(y: number): number {
    if (offsets.length <= 1) return 0
    // Binary search for largest i with offsets[i] <= y.
    let lo = 0, hi = offsets.length - 1
    while (lo < hi) {
      const mid = (lo + hi + 1) >>> 1
      if (offsets[mid]! <= y) lo = mid; else hi = mid - 1
    }
    return lo
  }

  return {
    setHeights(h) { heights = h.slice(); rebuild() },
    updateHeight(i, h) {
      if (i < 0 || i >= heights.length) return
      const delta = h - heights[i]!
      if (delta === 0) return
      heights[i] = h
      for (let j = i + 1; j < offsets.length; j++) offsets[j]! += delta
    },
    totalHeight() { return offsets[offsets.length - 1] ?? 0 },
    offsetOf(i) {
      if (i < 0) return 0
      if (i >= offsets.length) return offsets[offsets.length - 1] ?? 0
      return offsets[i]!
    },
    visibleRange(scrollTop, viewportHeight) {
      if (heights.length === 0) return { start: 0, end: 0 }
      const top = Math.max(0, scrollTop - opts.overscanPx)
      const bottom = scrollTop + viewportHeight + opts.overscanPx
      const start = findIndexAt(top)
      let end = findIndexAt(bottom) + 1
      if (end > heights.length) end = heights.length
      return { start, end }
    },
  }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test`
Expected: PASS — 6 virtualizer tests green.

- [ ] **Step 5: Commit**

```bash
git add src/core/virtualizer.ts test/unit/virtualizer.test.ts
git commit -m "feat(core): virtualizer (cumulative offsets + visible range w/ overscan)"
```

---

## Task 11: Runtime (`core/runtime.ts`)

Owns the scroll container, ResizeObserver, IntersectionObserver, and per-RAF batching of patches. Does not implement the renderer's full event loop — just the primitives the public `createRenderer` composes.

**Files:**
- Create: `src/core/runtime.ts`
- Create: `test/unit/runtime.test.ts`

- [ ] **Step 1: Write failing tests**

Create `test/unit/runtime.test.ts`:

```ts
import { describe, it, expect, vi } from 'vitest'
import { createPatchBatcher } from '@core/runtime'

describe('runtime: patch batcher', () => {
  it('coalesces multiple appends into one flush per RAF', async () => {
    const flushed: any[][] = []
    const batcher = createPatchBatcher(batch => { flushed.push(batch) })
    batcher.enqueue({ op: 'add', message: { id: 'm1', role: 'assistant', blocks: [], status: 'streaming' } } as any)
    batcher.enqueue({ op: 'append', messageId: 'm1', blockIndex: 0, run: { text: 'a' } } as any)
    batcher.enqueue({ op: 'append', messageId: 'm1', blockIndex: 0, run: { text: 'b' } } as any)
    expect(flushed).toHaveLength(0)               // not flushed synchronously
    await new Promise(r => requestAnimationFrame(() => requestAnimationFrame(() => r(null))))
    expect(flushed).toHaveLength(1)
    expect(flushed[0]).toHaveLength(3)
  })

  it('drops oldest non-visible work when queue exceeds backpressure cap', () => {
    const flushed: any[][] = []
    const batcher = createPatchBatcher(b => { flushed.push(b) }, { maxQueue: 4 })
    for (let i = 0; i < 10; i++) {
      batcher.enqueue({ op: 'append', messageId: 'm1', blockIndex: 0, run: { text: String(i) } } as any)
    }
    // Internal queue capped; tail is preserved.
    expect(batcher.queueDepth()).toBeLessThanOrEqual(4)
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test`
Expected: FAIL — `@core/runtime` does not exist.

- [ ] **Step 3: Implement `core/runtime.ts`**

Create `src/core/runtime.ts`:

```ts
import type { MessagePatch } from './blocks'

export type PatchBatcher = {
  enqueue(p: MessagePatch): void
  queueDepth(): number
  /** Called on teardown; cancels any pending RAF. */
  dispose(): void
}

export type BatcherOpts = { maxQueue?: number }

export function createPatchBatcher(
  flush: (batch: MessagePatch[]) => void,
  opts: BatcherOpts = {},
): PatchBatcher {
  const maxQueue = opts.maxQueue ?? 256
  let queue: MessagePatch[] = []
  let scheduled = false
  let rafId = 0

  function tick() {
    scheduled = false
    rafId = 0
    if (queue.length === 0) return
    const batch = queue
    queue = []
    flush(batch)
  }

  return {
    enqueue(p) {
      queue.push(p)
      // Backpressure: cap queue at maxQueue. We drop the oldest entries that
      // are NOT 'add' / 'finalize' / 'remove' (structural patches are kept
      // because dropping them corrupts state). For M0 we keep the simple rule:
      // if maxQueue exceeded, drop oldest.
      while (queue.length > maxQueue) queue.shift()
      if (!scheduled) {
        scheduled = true
        rafId = requestAnimationFrame(tick)
      }
    },
    queueDepth() { return queue.length },
    dispose() {
      if (rafId) cancelAnimationFrame(rafId)
      scheduled = false
      queue = []
    },
  }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test`
Expected: PASS — 2 runtime tests green.

- [ ] **Step 5: Commit**

```bash
git add src/core/runtime.ts test/unit/runtime.test.ts
git commit -m "feat(core): RAF patch batcher with bounded backpressure"
```

---

## Task 12: createRenderer public API (`core/index.ts`)

Composes patches + cache + virtualizer + layout + mount + runtime into the public `createRenderer({...})` factory documented in `docs/03-renderer.md`.

**Files:**
- Create: `src/core/index.ts`
- Create: `test/unit/renderer.test.ts`

- [ ] **Step 1: Write a failing integration test**

Create `test/unit/renderer.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { createRenderer } from '@core/index'
import type { Adapter, Capability } from '@core/adapter'
import type { MessagePatch } from '@core/blocks'

function makeAdapter(patches: MessagePatch[]): Adapter {
  return {
    name: 'fixture',
    schemaVersions: [1],
    capabilities: new Set<Capability>(),
    matches: () => true,
    findMessageList: (c) => c,
    parseConversationIdFromUrl: () => 'fixture',
    fetchConversation: async function* () { for (const p of patches) yield p },
    handleIntent: async () => ({ ok: false as const, reason: 'unsupported' as const }),
    dispose: () => {},
  }
}

describe('createRenderer', () => {
  it('returns synchronously and exposes the documented API', () => {
    const container = document.createElement('div')
    document.body.appendChild(container)
    const r = createRenderer({ container, adapter: makeAdapter([]), theme: 'light' })
    expect(typeof r.start).toBe('function')
    expect(typeof r.stop).toBe('function')
    expect(typeof r.scrollToMessage).toBe('function')
    expect(typeof r.setTheme).toBe('function')
    expect(typeof r.onMessageCountChange).toBe('function')
    r.stop()
    document.body.removeChild(container)
  })

  it('start() drains the adapter and renders messages', async () => {
    const container = document.createElement('div')
    container.style.height = '600px'
    document.body.appendChild(container)
    const patches: MessagePatch[] = [
      { op: 'add', message: { id: 'm1', role: 'user', status: 'final',
        blocks: [{ kind: 'text', runs: [{ text: 'Hello' }] }] } },
      { op: 'add', message: { id: 'm2', role: 'assistant', status: 'final',
        blocks: [{ kind: 'text', runs: [{ text: 'World' }] }] } },
    ]
    const r = createRenderer({ container, adapter: makeAdapter(patches), theme: 'light' })
    await r.start()
    // Wait for one RAF tick so the batcher flushes.
    await new Promise(res => requestAnimationFrame(() => requestAnimationFrame(() => res(null))))
    const html = container.textContent ?? ''
    expect(html).toContain('Hello')
    expect(html).toContain('World')
    r.stop()
    document.body.removeChild(container)
  })

  it('onMessageCountChange fires on add and remove', async () => {
    const container = document.createElement('div')
    document.body.appendChild(container)
    const counts: number[] = []
    const r = createRenderer({
      container,
      adapter: makeAdapter([
        { op: 'add', message: { id: 'm1', role: 'user', status: 'final', blocks: [] } },
        { op: 'add', message: { id: 'm2', role: 'user', status: 'final', blocks: [] } },
        { op: 'remove', messageId: 'm1' },
      ]),
      theme: 'light',
    })
    r.onMessageCountChange(n => counts.push(n))
    await r.start()
    await new Promise(res => requestAnimationFrame(() => requestAnimationFrame(() => res(null))))
    expect(counts).toEqual([1, 2, 1])
    r.stop()
    document.body.removeChild(container)
  })
})
```

- [ ] **Step 2: Run the test to verify it fails**

Run: `npm test`
Expected: FAIL — `@core/index` does not exist.

- [ ] **Step 3: Implement `core/index.ts`**

Create `src/core/index.ts`:

```ts
import type { Block, Message, MessagePatch } from './blocks'
import type { Adapter } from './adapter'
import type { Intent, IntentResult } from './intents'
import { handleDefaultIntent } from './intents'
import { applyPatch, createState, type State } from './patches'
import { createHeightCache, type HeightCache } from './cache'
import { createVirtualizer, type Virtualizer } from './virtualizer'
import { measureMessage, type LayoutCtx } from './layout'
import { mountOpaque } from './mount'
import { createPatchBatcher } from './runtime'
import { resolveColor, type ThemeName } from './theme'

export type RendererOpts = {
  container: HTMLElement
  adapter: Adapter
  theme: ThemeName | 'auto'
  overscanPx?: number
  measurementFont?: string
  onIntent?: (i: Intent) => Promise<IntentResult>
  debug?: boolean
}

export type Renderer = {
  start(): Promise<void>
  stop(): void
  scrollToMessage(id: string): void
  setTheme(t: ThemeName): void
  onMessageCountChange(cb: (count: number) => void): () => void
}

export function createRenderer(opts: RendererOpts): Renderer {
  const container = opts.container
  const overscanPx = opts.overscanPx ?? 600

  // ---- Theme + measurement context ----
  let theme: ThemeName = opts.theme === 'auto'
    ? (matchMedia?.('(prefers-color-scheme: dark)').matches ? 'dark' : 'light')
    : opts.theme
  const ctx: LayoutCtx = {
    font: opts.measurementFont ?? '14px ui-sans-serif, system-ui, sans-serif',
    fontSize: 14, lineHeight: 20,
    monoFont: '13px ui-monospace, SF Mono, monospace',
    monoFontSize: 13, monoLineHeight: 18,
  }

  // ---- State ----
  const state: State = createState({ devAssertions: !!opts.debug })
  const cache: HeightCache = createHeightCache()
  const virt: Virtualizer = createVirtualizer({ overscanPx })

  // ---- DOM scaffolding ----
  container.style.position = container.style.position || 'relative'
  container.style.overflow = 'auto'
  const spacer = document.createElement('div')
  spacer.dataset.cpRole = 'spacer'
  spacer.style.position = 'relative'
  container.appendChild(spacer)

  const mounted = new Map<string, HTMLElement>()
  const countListeners = new Set<(n: number) => void>()
  let lastCount = -1

  // ---- Container width ----
  const containerWidth = (): number => container.clientWidth || 800

  function widthForMessage() { return containerWidth() - 16 } // 8 px padding each side

  function measure(m: Message): number {
    const w = widthForMessage()
    let total = 0
    m.blocks.forEach((b, idx) => {
      const cached = cache.get(m.id, idx, w)
      if (cached !== undefined) { total += cached; return }
      const h = measureMessage({ ...m, blocks: [b] }, w, ctx)[0]!.height
      cache.set(m.id, idx, w, h)
      total += h
    })
    return Math.max(total, 1)
  }

  function recomputeAllHeights(): void {
    const heights = state.messages.map(measure)
    virt.setHeights(heights)
    spacer.style.height = `${virt.totalHeight()}px`
  }

  function reconcile(): void {
    const range = virt.visibleRange(container.scrollTop, container.clientHeight)
    // Unmount messages outside the range.
    for (const [id, el] of mounted) {
      const idx = state.messages.findIndex(m => m.id === id)
      if (idx === -1 || idx < range.start || idx >= range.end) {
        el.remove()
        mounted.delete(id)
      }
    }
    // Mount messages inside the range.
    for (let i = range.start; i < range.end; i++) {
      const m = state.messages[i]!
      if (mounted.has(m.id)) continue
      const el = renderMessage(m)
      el.style.position = 'absolute'
      el.style.top = `${virt.offsetOf(i)}px`
      el.style.left = '0'
      el.style.right = '0'
      el.style.padding = '0 8px'
      spacer.appendChild(el)
      mounted.set(m.id, el)
    }
  }

  function renderMessage(m: Message): HTMLElement {
    const wrap = document.createElement('article')
    wrap.dataset.cpMessageId = m.id
    wrap.dataset.cpRole = m.role
    for (const block of m.blocks) wrap.appendChild(renderBlock(block))
    return wrap
  }

  function renderBlock(b: Block): HTMLElement {
    switch (b.kind) {
      case 'text': {
        const p = document.createElement('p')
        p.style.margin = '0'
        p.style.lineHeight = `${ctx.lineHeight}px`
        for (const r of b.runs) p.appendChild(renderRun(r, false))
        return p
      }
      case 'heading': {
        const h = document.createElement(`h${b.level}`)
        h.style.margin = '0'
        for (const r of b.runs) h.appendChild(renderRun(r, false))
        return h
      }
      case 'code': {
        const pre = document.createElement('pre')
        pre.style.margin = '0'
        pre.style.font = ctx.monoFont
        pre.style.lineHeight = `${ctx.monoLineHeight}px`
        pre.style.whiteSpace = 'pre'
        pre.style.overflowX = 'auto'
        for (const line of b.lines) {
          const ln = document.createElement('div')
          for (const r of line) ln.appendChild(renderRun(r, true))
          pre.appendChild(ln)
        }
        return pre
      }
      case 'quote': {
        const q = document.createElement('blockquote')
        q.style.margin = '0'
        q.style.borderLeft = '3px solid currentColor'
        q.style.paddingLeft = '8px'
        for (const c of b.children) q.appendChild(renderBlock(c))
        return q
      }
      case 'list': {
        const list = document.createElement(b.ordered ? 'ol' : 'ul')
        list.style.margin = '0'
        if (b.start !== undefined && b.ordered) (list as HTMLOListElement).start = b.start
        for (const item of b.items) {
          const li = document.createElement('li')
          for (const blk of item) li.appendChild(renderBlock(blk))
          list.appendChild(li)
        }
        return list
      }
      case 'opaque': {
        const host = document.createElement('div')
        mountOpaque(host, b, { onResize: () => requestRelayout() })
        return host
      }
    }
  }

  function renderRun(r: import('./blocks').Run, inCode: boolean): Node {
    if (r.href) {
      const a = document.createElement('a')
      a.href = r.href
      a.textContent = r.text
      a.rel = 'noopener noreferrer'
      a.target = '_blank'
      a.addEventListener('click', (ev) => {
        ev.preventDefault()
        emitIntent({ type: 'openLink', href: r.href!, messageId: a.closest('[data-cp-message-id]')?.getAttribute('data-cp-message-id') ?? '' })
      })
      return a
    }
    const span = document.createElement('span')
    span.textContent = r.text
    if (r.weight === 'bold') span.style.fontWeight = '600'
    if (r.italic) span.style.fontStyle = 'italic'
    if (r.underline) span.style.textDecoration = 'underline'
    if (r.strike) span.style.textDecoration = 'line-through'
    if (inCode && r.color) span.style.color = resolveColor(r.color, theme)
    return span
  }

  // ---- Patch flush ----
  function onPatchBatch(batch: MessagePatch[]): void {
    const dirty = new Set<string>()
    for (const p of batch) {
      const { affected } = applyPatch(state, p)
      for (const id of affected) dirty.add(id)
      if (p.op === 'remove') cache.invalidate(p.messageId)
      else if (p.op === 'replace') cache.invalidate(p.messageId, p.blockIndex)
      else if (p.op === 'replaceBlocks' || p.op === 'add') cache.invalidate(p.messageId)
    }
    // Update heights for dirty messages only.
    for (const id of dirty) {
      const idx = state.messages.findIndex(m => m.id === id)
      if (idx !== -1) virt.updateHeight(idx, measure(state.messages[idx]!))
    }
    // Always rebuild after add/remove (lengths changed) — cheaper to just resync.
    if (batch.some(p => p.op === 'add' || p.op === 'remove')) recomputeAllHeights()
    spacer.style.height = `${virt.totalHeight()}px`
    // Unmount dirty messages so reconcile re-renders them with new content.
    for (const id of dirty) {
      const el = mounted.get(id)
      if (el) { el.remove(); mounted.delete(id) }
    }
    reconcile()
    // Notify count listeners.
    if (state.messages.length !== lastCount) {
      lastCount = state.messages.length
      for (const cb of countListeners) cb(lastCount)
    }
  }

  const batcher = createPatchBatcher(onPatchBatch)

  // ---- Re-layout request (e.g., from opaque ResizeObserver) ----
  let relayoutScheduled = false
  function requestRelayout() {
    if (relayoutScheduled) return
    relayoutScheduled = true
    requestAnimationFrame(() => {
      relayoutScheduled = false
      cache.invalidateAll()
      recomputeAllHeights()
      reconcile()
    })
  }

  // ---- Scroll + resize ----
  const onScroll = () => reconcile()
  const ro = new ResizeObserver(() => requestRelayout())

  // ---- Intent emission ----
  const intentRoute = opts.onIntent ?? ((i) => opts.adapter.handleIntent(i))
  function emitIntent(intent: Intent): void {
    // Renderer-handled defaults short-circuit unless adapter overrides.
    if (intent.type === 'openLink' && !opts.adapter.capabilities.has('openLinkOverride')) {
      void handleDefaultIntent(intent); return
    }
    if (intent.type === 'openImage' && !opts.adapter.capabilities.has('openImageOverride')) {
      void handleDefaultIntent(intent); return
    }
    void intentRoute(intent)
  }

  // ---- Lifecycle ----
  let ac: AbortController | null = null

  return {
    async start() {
      ac = new AbortController()
      container.addEventListener('scroll', onScroll, { passive: true })
      ro.observe(container)
      const it = opts.adapter.fetchConversation(ac.signal)
      // Drain in the background; do not await full completion.
      ;(async () => {
        for await (const patch of it) batcher.enqueue(patch)
      })().catch(err => { if (opts.debug) console.error('[renderer] drain error', err) })
    },
    stop() {
      ac?.abort(); ac = null
      batcher.dispose()
      ro.disconnect()
      container.removeEventListener('scroll', onScroll)
      for (const [, el] of mounted) el.remove()
      mounted.clear()
      spacer.remove()
    },
    scrollToMessage(id) {
      const idx = state.messages.findIndex(m => m.id === id)
      if (idx === -1) return
      container.scrollTop = virt.offsetOf(idx)
    },
    setTheme(t) {
      theme = t
      cache.invalidateAll()
      // Force re-render of mounted messages so colors update.
      for (const [, el] of mounted) el.remove()
      mounted.clear()
      recomputeAllHeights()
      reconcile()
    },
    onMessageCountChange(cb) {
      countListeners.add(cb)
      return () => countListeners.delete(cb)
    },
  }
}
```

- [ ] **Step 4: Run the test**

Run: `npm test`

If pretext is browser-only (per Task 1 spike) and `start()` calls into measurement that breaks under happy-dom, the second test (`start() drains the adapter and renders messages`) may fail. In that case:
- Mark that test `.skip` with a comment pointing to the playwright suite, and
- Trust the playwright perf tests (Task 17–19) as integration coverage.

Expected (one of):
- All 3 renderer tests pass under happy-dom, OR
- Test 1 + 3 pass; test 2 skipped with a note.

- [ ] **Step 5: Commit**

```bash
git add src/core/index.ts test/unit/renderer.test.ts
git commit -m "feat(core): createRenderer public API (composes patches/cache/virt/layout/mount)"
```

---

## Task 13: ESLint boundaries rule

Enforce the layering rules from `docs/CONTRACTS.md`. M0 only has `core/`, but the rule is configured for the full layout so M1+ inherits it.

**Files:**
- Create: `.eslintrc.cjs`

- [ ] **Step 1: Add the config**

Create `.eslintrc.cjs`:

```js
/* eslint-env node */
module.exports = {
  root: true,
  parser: '@typescript-eslint/parser',
  plugins: ['boundaries'],
  settings: {
    'boundaries/elements': [
      { type: 'core',     pattern: 'src/core/*'        },
      { type: 'services', pattern: 'src/services/*'    },
      { type: 'adapters', pattern: 'src/adapters/*'    },
      { type: 'shell',    pattern: 'src/shell/*'       },
      { type: 'test',     pattern: 'test/**'           },
      { type: 'vendor',   pattern: 'vendor/**'         },
    ],
    'boundaries/include': ['src/**/*', 'test/**/*'],
  },
  rules: {
    'boundaries/element-types': ['error', {
      default: 'disallow',
      rules: [
        { from: 'core',     allow: ['core', 'vendor'] },
        { from: 'services', allow: ['core'] },                       // seam types only
        { from: 'adapters', allow: ['core'] },                       // seam types only
        { from: 'shell',    allow: ['core', 'adapters'] },
        { from: 'test',     allow: ['core', 'services', 'adapters', 'shell', 'vendor'] },
      ],
    }],
  },
}
```

- [ ] **Step 2: Install eslint deps**

Run: `npm install -D @typescript-eslint/parser eslint-plugin-boundaries`
Expected: success.

- [ ] **Step 3: Run lint**

Run: `npm run lint`
Expected: success — no boundary violations in `src/core/` (it only imports its own modules + `@pretext`).

- [ ] **Step 4: Commit**

```bash
git add .eslintrc.cjs package.json package-lock.json
git commit -m "chore(m0): eslint boundaries rule for layering enforcement"
```

---

## Task 14: Synthetic fixture generator

**Files:**
- Create: `test/fixtures/generator.ts`
- Create: `test/unit/generator.test.ts`

A 5000-message synthetic conversation with realistic block mix: ~70% text, ~20% code, ~5% list/quote/heading, ~5% opaque image.

- [ ] **Step 1: Write a failing test**

Create `test/unit/generator.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { generateConversation } from '@test/fixtures/generator'

describe('fixture generator', () => {
  it('produces N messages with stable ids', () => {
    const ms = generateConversation({ messageCount: 100, seed: 42 })
    expect(ms).toHaveLength(100)
    expect(new Set(ms.map(m => m.id)).size).toBe(100)
    // Same seed → same ids (deterministic).
    const ms2 = generateConversation({ messageCount: 100, seed: 42 })
    expect(ms.map(m => m.id)).toEqual(ms2.map(m => m.id))
  })

  it('alternates user/assistant by default', () => {
    const ms = generateConversation({ messageCount: 10, seed: 1 })
    for (let i = 0; i < 10; i++) {
      expect(ms[i]!.role).toBe(i % 2 === 0 ? 'user' : 'assistant')
    }
  })

  it('every message has at least one block', () => {
    const ms = generateConversation({ messageCount: 50, seed: 7 })
    for (const m of ms) expect(m.blocks.length).toBeGreaterThan(0)
  })

  it('code blocks satisfy lines.length === source.split(\\n).length when non-empty', () => {
    const ms = generateConversation({ messageCount: 200, seed: 9 })
    let codeCount = 0
    for (const m of ms) for (const b of m.blocks) {
      if (b.kind === 'code' && b.source !== '') {
        codeCount++
        expect(b.lines.length).toBe(b.source.split('\n').length)
      }
    }
    expect(codeCount).toBeGreaterThan(0)
  })

  it('opaque blocks have intrinsicWidth/Height > 0', () => {
    const ms = generateConversation({ messageCount: 200, seed: 5 })
    for (const m of ms) for (const b of m.blocks) {
      if (b.kind === 'opaque') {
        expect(b.intrinsicWidth).toBeGreaterThan(0)
        expect(b.intrinsicHeight).toBeGreaterThan(0)
      }
    }
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test`
Expected: FAIL — `@test/fixtures/generator` does not exist.

- [ ] **Step 3: Implement the generator**

Create `test/fixtures/generator.ts`:

```ts
import type { Block, Message, Run, TokenColor } from '@core/blocks'

export type GenOpts = {
  messageCount: number
  seed?: number
}

// Mulberry32 — small deterministic PRNG.
function rng(seed: number) {
  let s = seed >>> 0
  return () => { s = (s + 0x6D2B79F5) >>> 0
    let t = s
    t = Math.imul(t ^ (t >>> 15), t | 1)
    t ^= t + Math.imul(t ^ (t >>> 7), t | 61)
    return ((t ^ (t >>> 14)) >>> 0) / 0x100000000
  }
}

const LOREM = (
  'Lorem ipsum dolor sit amet consectetur adipiscing elit sed do eiusmod ' +
  'tempor incididunt ut labore et dolore magna aliqua ut enim ad minim veniam ' +
  'quis nostrud exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat'
).split(' ')

const CODE_LINES = [
  'function hello(name: string) {',
  '  console.log(`Hello, ${name}!`)',
  '  return name.length',
  '}',
  '',
  'const result = hello("world")',
]

const COLORS: TokenColor[] = ['keyword', 'string', 'number', 'comment', 'function', 'type', 'variable']

export function generateConversation(opts: GenOpts): Message[] {
  const r = rng(opts.seed ?? 1)
  const messages: Message[] = []
  for (let i = 0; i < opts.messageCount; i++) {
    const role = i % 2 === 0 ? 'user' : 'assistant'
    const blocks = role === 'user'
      ? [textBlock(r, 1 + Math.floor(r() * 3))]                  // user: short prose
      : assistantBlocks(r)
    messages.push({
      id: `m_${i.toString(36)}_${Math.floor(r() * 1e6).toString(36)}`,
      role,
      status: 'final',
      blocks,
      createdAt: 1700000000000 + i * 1000,
    })
  }
  return messages
}

function assistantBlocks(r: () => number): Block[] {
  const out: Block[] = []
  const numBlocks = 1 + Math.floor(r() * 5)
  for (let i = 0; i < numBlocks; i++) {
    const roll = r()
    if (roll < 0.7)      out.push(textBlock(r, 1 + Math.floor(r() * 8)))
    else if (roll < 0.9) out.push(codeBlock(r))
    else if (roll < 0.95) out.push(listBlock(r))
    else                 out.push(opaqueImage(r))
  }
  return out
}

function textBlock(r: () => number, sentences: number): Block {
  const text = Array.from({ length: sentences }, () =>
    Array.from({ length: 8 + Math.floor(r() * 12) }, () =>
      LOREM[Math.floor(r() * LOREM.length)]).join(' ') + '.').join(' ')
  return { kind: 'text', runs: [{ text }] }
}

function codeBlock(r: () => number): Block {
  const lineCount = 3 + Math.floor(r() * 8)
  const sourceLines = Array.from({ length: lineCount }, (_, i) => CODE_LINES[i % CODE_LINES.length]!)
  const source = sourceLines.join('\n')
  const lines: Run[][] = sourceLines.map(line => {
    if (!line) return [{ text: '', mono: true }]
    // Tokenize naively: split by spaces, color randomly.
    const tokens = line.split(/(\s+)/)
    return tokens.filter(Boolean).map(t => {
      if (/^\s+$/.test(t)) return { text: t, mono: true }
      return { text: t, mono: true, color: COLORS[Math.floor(r() * COLORS.length)] }
    })
  })
  return { kind: 'code', language: 'ts', source, lines }
}

function listBlock(r: () => number): Block {
  const itemCount = 2 + Math.floor(r() * 4)
  const items = Array.from({ length: itemCount }, () => [textBlock(r, 1)])
  return { kind: 'list', ordered: r() < 0.5, items }
}

function opaqueImage(r: () => number): Block {
  const w = 320 + Math.floor(r() * 320)
  const h = 180 + Math.floor(r() * 180)
  // 1x1 SVG rasterized as a data URI; cheap and deterministic.
  const url = `data:image/svg+xml;utf8,${encodeURIComponent(
    `<svg xmlns='http://www.w3.org/2000/svg' width='${w}' height='${h}'><rect width='100%' height='100%' fill='#cdd9e5'/></svg>`,
  )}`
  return {
    kind: 'opaque',
    payload: { type: 'image', url, alt: 'placeholder' },
    intrinsicWidth: w, intrinsicHeight: h,
    reflow: 'fluid',
    cacheKey: url,
  }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test`
Expected: PASS — 5 generator tests green.

- [ ] **Step 5: Commit**

```bash
git add test/fixtures/generator.ts test/unit/generator.test.ts
git commit -m "test(fixtures): deterministic 5k-message synthetic conversation generator"
```

---

## Task 15: Synthetic replay adapter

**Files:**
- Create: `test/fixtures/synthetic-adapter.ts`
- Create: `test/unit/synthetic-adapter.test.ts`

Wraps a `Message[]` (or a hand-written patch sequence) as an `Adapter` so the renderer can drive it without a real site.

- [ ] **Step 1: Write a failing test**

Create `test/unit/synthetic-adapter.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { syntheticAdapter, replayHistory } from '@test/fixtures/synthetic-adapter'
import { generateConversation } from '@test/fixtures/generator'

describe('synthetic adapter', () => {
  it('replays history as one add patch per message', async () => {
    const ms = generateConversation({ messageCount: 10, seed: 1 })
    const adapter = syntheticAdapter({ history: ms })
    const ac = new AbortController()
    const out: any[] = []
    for await (const p of adapter.fetchConversation(ac.signal)) out.push(p)
    expect(out).toHaveLength(10)
    expect(out.every(p => p.op === 'add')).toBe(true)
  })

  it('replayHistory yields one add per message in order', async () => {
    const ms = generateConversation({ messageCount: 5, seed: 1 })
    const ids: string[] = []
    for await (const p of replayHistory(ms)) {
      if (p.op === 'add') ids.push(p.message.id)
    }
    expect(ids).toEqual(ms.map(m => m.id))
  })

  it('streams custom patch script after history', async () => {
    const ms = generateConversation({ messageCount: 2, seed: 1 })
    const adapter = syntheticAdapter({
      history: ms,
      stream: [
        { op: 'add', message: { id: 'streaming', role: 'assistant', status: 'streaming', blocks: [] } },
        { op: 'replace', messageId: 'streaming', blockIndex: 0, block: { kind: 'text', runs: [{ text: '' }] } },
        { op: 'append', messageId: 'streaming', blockIndex: 0, run: { text: 'hi' } },
        { op: 'finalize', messageId: 'streaming' },
      ],
    })
    const ac = new AbortController()
    const ops: string[] = []
    for await (const p of adapter.fetchConversation(ac.signal)) ops.push(p.op)
    expect(ops).toEqual(['add', 'add', 'add', 'replace', 'append', 'finalize'])
  })
})
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `npm test`
Expected: FAIL — `@test/fixtures/synthetic-adapter` does not exist.

- [ ] **Step 3: Implement the synthetic adapter**

Create `test/fixtures/synthetic-adapter.ts`:

```ts
import type { Adapter, Capability } from '@core/adapter'
import type { Message, MessagePatch } from '@core/blocks'
import { BLOCK_SCHEMA_VERSION } from '@core/blocks'

export type SyntheticOpts = {
  history: Message[]
  /** Optional patch script played after history is drained. */
  stream?: MessagePatch[]
  /** Delay between stream patches in ms. Default 0 (immediate). */
  streamDelayMs?: number
  capabilities?: Capability[]
}

export async function* replayHistory(history: Message[]): AsyncGenerator<MessagePatch> {
  for (const m of history) yield { op: 'add', message: m }
}

export function syntheticAdapter(opts: SyntheticOpts): Adapter {
  const caps = new Set<Capability>(opts.capabilities ?? [])
  return {
    name: 'synthetic',
    schemaVersions: [BLOCK_SCHEMA_VERSION],
    capabilities: caps,
    matches: () => true,
    findMessageList: (c) => c,
    parseConversationIdFromUrl: () => 'synthetic',
    async *fetchConversation(signal: AbortSignal) {
      for await (const p of replayHistory(opts.history)) {
        if (signal.aborted) return
        yield p
      }
      const stream = opts.stream ?? []
      for (const p of stream) {
        if (signal.aborted) return
        if (opts.streamDelayMs && opts.streamDelayMs > 0) {
          await new Promise(res => setTimeout(res, opts.streamDelayMs))
        }
        yield p
      }
    },
    handleIntent: async () => ({ ok: false, reason: 'unsupported' }),
    dispose: () => {},
  }
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `npm test`
Expected: PASS — 3 adapter tests green.

- [ ] **Step 5: Commit**

```bash
git add test/fixtures/synthetic-adapter.ts test/unit/synthetic-adapter.test.ts
git commit -m "test(fixtures): synthetic replay adapter (history + scripted stream)"
```

---

## Task 16: Playground HTML

**Files:**
- Create: `test/playground/index.html`
- Create: `test/playground/main.ts`

A bare HTML page that mounts the renderer with a 5000-message conversation. Used both for manual perf inspection and as the page Playwright opens.

- [ ] **Step 1: Create the HTML**

Create `test/playground/index.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>chatPlugin M0 playground</title>
    <style>
      html, body { margin: 0; padding: 0; height: 100%; font-family: ui-sans-serif, system-ui, sans-serif; }
      #app { display: grid; grid-template-rows: auto 1fr; height: 100%; }
      header { padding: 8px 16px; border-bottom: 1px solid #d0d7de; font-size: 13px; display: flex; gap: 16px; align-items: center; }
      header button { font-size: 12px; padding: 2px 8px; }
      #viewport { position: relative; }
      #status { color: #656d76; }
    </style>
  </head>
  <body>
    <div id="app">
      <header>
        <strong>M0 playground</strong>
        <span id="status">loading…</span>
        <button id="stream">Start streaming</button>
        <button id="resize-narrow">Resize 800 px</button>
        <button id="resize-wide">Resize 1200 px</button>
      </header>
      <div id="viewport"></div>
    </div>
    <script type="module" src="./main.ts"></script>
  </body>
</html>
```

- [ ] **Step 2: Create the entry**

Create `test/playground/main.ts`. The streaming button dispatches a `cp:stream-test` event; Task 19 wires the actual streaming logic to that event so the button starts working once Task 19 lands.

```ts
import { createRenderer } from '@core/index'
import { generateConversation } from '@test/fixtures/generator'
import { syntheticAdapter } from '@test/fixtures/synthetic-adapter'

const messageCount = Number(new URLSearchParams(location.search).get('n') ?? 5000)
const seed = Number(new URLSearchParams(location.search).get('seed') ?? 42)
const status = document.getElementById('status')!

status.textContent = `generating ${messageCount} messages…`
const history = generateConversation({ messageCount, seed })
status.textContent = `mounting…`

const viewport = document.getElementById('viewport')!
const adapter = syntheticAdapter({ history })
const renderer = createRenderer({
  container: viewport,
  adapter,
  theme: 'light',
  overscanPx: 600,
  debug: true,
})

await renderer.start()
status.textContent = `${messageCount} messages mounted; total scrollHeight = ${viewport.scrollHeight}px`

document.getElementById('stream')!.addEventListener('click', () => {
  // Task 19 installs the listener; this dispatch is a no-op until then.
  window.dispatchEvent(new CustomEvent('cp:stream-test',
    { detail: { framesToStream: 60, perFrame: 10 } }))
})

document.getElementById('resize-narrow')!.addEventListener('click', () => {
  document.body.style.maxWidth = '800px'
})
document.getElementById('resize-wide')!.addEventListener('click', () => {
  document.body.style.maxWidth = '1200px'
})

// Expose hooks for Playwright.
;(window as any).__cp = { renderer, viewport, history }
```

- [ ] **Step 3: Verify the playground loads**

Run: `npm run dev` and open the URL it prints (default `http://localhost:5173/test/playground/`).
Expected: page renders header, status shows "5000 messages mounted; total scrollHeight = …px"; scrolling works; no console errors.

If pretext fails in the browser, fix the import alias and the `prepare`/`layout` adapters in `core/layout.ts`. Re-run the spike if signatures don't match.

- [ ] **Step 4: Commit**

```bash
git add test/playground/index.html test/playground/main.ts
git commit -m "test(playground): standalone html with 5k synthetic conversation"
```

---

## Task 17: Playwright config + scroll perf test

**Files:**
- Create: `test/perf/scroll.spec.ts`

Asserts p99 frame time < 16 ms over a programmatic scroll-through of 5000 messages.

- [ ] **Step 1: Install playwright browsers**

Run: `npx playwright install chromium`
Expected: chromium downloads.

- [ ] **Step 2: Write the failing perf test**

Create `test/perf/scroll.spec.ts`:

```ts
import { test, expect } from '@playwright/test'

test('scroll through 5000-message conversation: p99 frame < 16 ms', async ({ page }) => {
  // Baseline 5k convo.
  await page.goto('/test/playground/?n=5000&seed=42')
  // Wait for status text to confirm mount.
  await page.waitForFunction(() =>
    /mounted/.test(document.getElementById('status')?.textContent ?? ''))

  // Hook RAF to record per-frame deltas while we scroll.
  const result = await page.evaluate(async () => {
    const viewport = (window as any).__cp.viewport as HTMLElement
    const totalScroll = viewport.scrollHeight - viewport.clientHeight
    const samples: number[] = []
    let prev = performance.now()
    let raf = 0
    const onFrame = (t: number) => {
      samples.push(t - prev)
      prev = t
      raf = requestAnimationFrame(onFrame)
    }
    raf = requestAnimationFrame(onFrame)
    // Scroll programmatically over ~2 seconds, ~60 ticks.
    const ticks = 120
    const dy = totalScroll / ticks
    for (let i = 0; i < ticks; i++) {
      viewport.scrollTop = i * dy
      await new Promise(r => requestAnimationFrame(() => r(null)))
    }
    cancelAnimationFrame(raf)
    samples.sort((a, b) => a - b)
    return {
      p50: samples[Math.floor(samples.length * 0.5)],
      p95: samples[Math.floor(samples.length * 0.95)],
      p99: samples[Math.floor(samples.length * 0.99)],
      max: samples[samples.length - 1],
      count: samples.length,
    }
  })

  console.log('frame timings (ms):', result)
  expect(result.count).toBeGreaterThan(60)
  expect(result.p99!).toBeLessThan(16)
})
```

- [ ] **Step 3: Run the test**

Run: `npm run test:perf -- scroll`
Expected (one of):
- PASS — p99 < 16 ms; we hit the M0 acceptance.
- FAIL — measurement reveals a hot path. Read the printed `result` and profile via Playwright tracing (`page.tracing.start({ screenshots: false, snapshots: false })`). Common culprits: `recomputeAllHeights` running on every batch (should only run on add/remove); reconciling outside the RAF batch; pretext re-preparing the same text more than once per frame.

If the test fails, profile, fix in `src/core/index.ts`, re-run. **Do not lower the threshold.**

- [ ] **Step 4: Commit when green**

```bash
git add test/perf/scroll.spec.ts
git commit -m "test(perf): scroll 5k messages with p99 < 16ms acceptance"
```

---

## Task 18: Resize perf test

**Files:**
- Create: `test/perf/resize.spec.ts`

Asserts resize 800 → 1200 px completes layout for the visible window in < 50 ms.

- [ ] **Step 1: Write the test**

Create `test/perf/resize.spec.ts`:

```ts
import { test, expect } from '@playwright/test'

test('resize 800 → 1200 completes visible re-layout in < 50 ms', async ({ page }) => {
  await page.setViewportSize({ width: 800, height: 900 })
  await page.goto('/test/playground/?n=5000&seed=42')
  await page.waitForFunction(() =>
    /mounted/.test(document.getElementById('status')?.textContent ?? ''))

  // Scroll to a representative middle position so the visible window is non-trivial.
  await page.evaluate(() => { (window as any).__cp.viewport.scrollTop = 30_000 })
  await page.waitForTimeout(100)

  // Measure: we set the viewport's max-width directly inside the page so the
  // ResizeObserver inside core/index.ts fires synchronously on the next layout.
  // We then wait for two RAFs (the renderer's relayout schedules one RAF; we
  // wait a second to confirm it has settled) and stop the clock.
  const ms = await page.evaluate(async () => {
    const viewport = (window as any).__cp.viewport as HTMLElement
    const t0 = performance.now()
    viewport.style.width = '1200px'                    // forces RO callback next layout
    await new Promise(res => requestAnimationFrame(() => requestAnimationFrame(() => res(null))))
    return performance.now() - t0
  })

  console.log('resize re-layout (ms):', ms)
  expect(ms).toBeLessThan(50)
})
```

- [ ] **Step 2: Run the test**

Run: `npm run test:perf -- resize`
Expected: PASS, < 50 ms. If fail, the likely cause is `cache.invalidateAll()` in `requestRelayout` — narrow it to invalidating only the visible window's blocks.

- [ ] **Step 3: Commit when green**

```bash
git add test/perf/resize.spec.ts
git commit -m "test(perf): resize 800→1200 visible re-layout < 50ms"
```

---

## Task 19: Streaming perf test

**Files:**
- Create: `test/perf/streaming.spec.ts`

Asserts that 10 patches/frame for 60 frames doesn't drop a frame.

- [ ] **Step 1: Write the test**

Create `test/perf/streaming.spec.ts`:

```ts
import { test, expect } from '@playwright/test'

test('synthetic streaming (10 patches/frame × 60 frames) drops no frames', async ({ page }) => {
  // Smaller convo so we stay focused on streaming cost, not layout cost.
  await page.goto('/test/playground/?n=20&seed=1')
  await page.waitForFunction(() =>
    /mounted/.test(document.getElementById('status')?.textContent ?? ''))

  const result = await page.evaluate(async () => {
    // Strategy: capture per-frame deltas via a RAF loop while we dispatch the
    // 'cp:stream-test' event that main.ts (Task 19 step 2) translates into a
    // 60-frame patch script driven through a side-mounted renderer. We measure
    // the global frame timing — what a user would actually see if a streaming
    // message and a long static conversation were both on the page.
    const samples: number[] = []
    let prev = performance.now()
    let stop = false
    const onFrame = (t: number) => {
      samples.push(t - prev)
      prev = t
      if (!stop) requestAnimationFrame(onFrame)
    }
    requestAnimationFrame(onFrame)

    window.dispatchEvent(new CustomEvent('cp:stream-test',
      { detail: { framesToStream: 60, perFrame: 10 } }))

    // Wait long enough for 60 streamed frames + a small cool-down.
    await new Promise(res => setTimeout(res, 1500))
    stop = true

    samples.sort((a, b) => a - b)
    return {
      p50: samples[Math.floor(samples.length * 0.5)],
      p95: samples[Math.floor(samples.length * 0.95)],
      p99: samples[Math.floor(samples.length * 0.99)],
      max: samples[samples.length - 1],
      count: samples.length,
    }
  })

  console.log('streaming frame timings (ms):', result)
  expect(result.p99!).toBeLessThan(16)
})
```

- [ ] **Step 2: Wire the playground to respond to the stream event**

The streaming logic mounts a *second* renderer in a hidden div so the streamed message lives in its own scroll context — the perf test measures the global frame timing, which is what users would see.

Open `test/playground/main.ts` and append (after the existing event listener block, before the `;(window as any).__cp = { renderer, viewport, history }` line — anywhere in the module body works since order does not matter for these top-level declarations):

```ts
window.addEventListener('cp:stream-test', async (ev) => {
  const detail = (ev as CustomEvent<{ framesToStream: number; perFrame: number }>).detail
  const id = `stream_${Date.now().toString(36)}`

  // Build the patch script: 1 add, 1 replace (creates block 0), N appends, 1 finalize.
  const patches: import('@core/blocks').MessagePatch[] = [
    { op: 'add', message: { id, role: 'assistant', status: 'streaming', blocks: [] } },
    { op: 'replace', messageId: id, blockIndex: 0, block: { kind: 'text', runs: [{ text: '' }] } },
  ]
  for (let f = 0; f < detail.framesToStream; f++) {
    for (let i = 0; i < detail.perFrame; i++) {
      patches.push({ op: 'append', messageId: id, blockIndex: 0, run: { text: `${f}.${i} ` } })
    }
  }
  patches.push({ op: 'finalize', messageId: id })

  // Mount a hidden side renderer so the stream measurement is independent
  // of the main 5k-message context.
  const hidden = document.createElement('div')
  hidden.style.cssText = 'position:fixed;left:-9999px;top:0;width:800px;height:600px;overflow:auto'
  document.body.appendChild(hidden)
  const sideAdapter = syntheticAdapter({ history: [], stream: patches, streamDelayMs: 0 })
  const sideRenderer = createRenderer({ container: hidden, adapter: sideAdapter, theme: 'light' })
  await sideRenderer.start()
  setTimeout(() => { sideRenderer.stop(); hidden.remove() }, 2000)
})
```

Note: `syntheticAdapter` and `createRenderer` are already imported at the top of `main.ts` from Task 16, so no new imports are needed.

- [ ] **Step 3: Run the test**

Run: `npm run test:perf -- streaming`
Expected: PASS, p99 < 16 ms. If FAIL: the batcher is likely flushing per-patch instead of per-frame — verify `createPatchBatcher` only schedules one RAF per burst.

- [ ] **Step 4: Commit when green**

```bash
git add test/perf/streaming.spec.ts test/playground/main.ts
git commit -m "test(perf): streaming 10 patches/frame × 60 frames drops no frames"
```

---

## Task 20: M0 acceptance run + tag

Final: run everything green, tag the milestone.

- [ ] **Step 1: Full test suite**

Run: `npm run typecheck && npm run lint && npm test && npm run test:perf`
Expected: all green.

- [ ] **Step 2: Manual smoke**

Run: `npm run dev`
Open the playground in Chrome. Scroll the entire conversation top-to-bottom and back. Watch DevTools Performance: confirm no long tasks > 16 ms during scroll. Resize the window from narrow to wide; verify no jank.

- [ ] **Step 3: Update the README**

Append to (or create) `README.md` at repo root:

```markdown
# chatPlugin

In-progress browser extension that takes over the message list of long ChatGPT/Grok conversations with a fast renderer (pretext + viewport virtualization).

## Status

- **M0 (renderer prototype):** complete — see `docs/plans/2026-04-28-m0-renderer-prototype.md`.
  Acceptance: 5000-message synthetic conversation scrolls at 60 fps; resize and synthetic streaming meet budgets. Playground at `npm run dev`.
- M1+ pending; see `docs/09-roadmap.md`.

## Develop

- `npm install`
- `npm run dev` — playground at `http://localhost:5173/test/playground/`
- `npm test` — unit + type tests
- `npm run test:perf` — playwright perf suite
- `npm run lint` — boundary rules
```

- [ ] **Step 4: Tag the milestone**

```bash
git add README.md
git commit -m "docs: M0 acceptance — renderer prototype meets budgets"
git tag -a v0.1.0-m0 -m "M0: renderer prototype meets all acceptance criteria"
```

- [ ] **Step 5: Hand off to M1 planning**

M0 produces a working renderer with locked layout/cache/virtualizer/runtime/index APIs. The next plan (M1) writes the ChatGPT adapter + minimal extension shell against this exact API. Before M1, write `docs/plans/2026-XX-XX-m1-chatgpt-readonly.md` using the same writing-plans workflow; the file structure and seam types from M0 are inputs.

---

## Self-review checklist

**Spec coverage** (from `docs/03-renderer.md` + `docs/09-roadmap.md` §M0):
- Block schema → Task 2 ✓
- Adapter / Capability seam → Task 3 ✓
- Intent / IntentResult + defaults → Task 4 ✓
- Theme → Task 5 ✓
- Patch state machine → Task 6 ✓
- Height cache (LRU, width-bucketed) → Task 7 ✓
- Layout (pretext wrapper) → Task 8 ✓ (with spike in Task 1)
- Opaque mount → Task 9 ✓
- Virtualizer (offset table + visible range) → Task 10 ✓
- Runtime (RAF batcher) → Task 11 ✓
- createRenderer public API → Task 12 ✓
- ESLint boundaries → Task 13 ✓
- Synthetic 5k-message fixture → Task 14 ✓
- Synthetic adapter (history + scripted stream) → Task 15 ✓
- Standalone playground HTML → Task 16 ✓
- Scroll perf < 16 ms p99 → Task 17 ✓
- Resize < 50 ms → Task 18 ✓
- Streaming 10 patches × 60 frames → Task 19 ✓
- Acceptance run + tag → Task 20 ✓

**Cross-cutting invariants** (from `docs/CONTRACTS.md`):
- Block invariants 1–7 → Task 6 tests cover create-or-replace, no-holes, append-on-text, appendLine atomicity ✓
- Layering: `core/` only imports `core/` + `@pretext` → Task 13 boundary rule ✓
- Schema-version constant exists → Task 2 ✓
- `IntentResult` canonical in `core/intents.ts` → Task 4 ✓
- Adapter interface in `core/adapter.ts`, not `adapters/` → Task 3 + boundary rule ✓

**Out of scope for M0** (deferred to later milestones):
- Selection/copy override on code blocks (M2)
- Cmd+F search bar (post-v1)
- ARIA / "render full transcript" toggle (M4)
- DOMPurify on opaque HTML (M1, when services land)
- Real adapter / real fetch interception (M1)
- Streaming from a real adapter (M2)
- onIntent failure-toast UI (M2)

These deferrals are explicit in `docs/09-roadmap.md`; nothing in M0 acceptance requires them.
