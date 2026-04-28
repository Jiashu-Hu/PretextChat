# Layer 1: Renderer (`src/core/`)

Universal, site-agnostic, dependency-light. Knows nothing about ChatGPT, Grok, markdown, or syntax languages. Consumes `Block`s and `MessagePatch`es; produces a fast scrolling viewport.

## Responsibilities

- Lay out text and code blocks with pretext.
- Mount opaque blocks at known intrinsic dimensions.
- Maintain a height cache keyed by `(messageId, blockIndex, width)`.
- Window the DOM to the viewport ± overscan.
- Handle scroll restoration, resize, theme switch, font-load.
- Emit `Intent`s for user actions; never call adapter functions directly.

## Public API

```ts
import { createRenderer } from 'chatPlugin/core'

const renderer = createRenderer({
  container: HTMLElement,         // mount point inside the page
  adapter: Adapter,               // see 04-adapters.md
  theme: 'light' | 'dark' | 'auto',
  overscanPx?: number,            // default 600
  measurementFont?: CanvasFont,   // default: derived from container CSS
  onIntent?: (i: Intent) => Promise<IntentResult>, // awaited; default forwards to adapter.handleIntent
  debug?: boolean,
})

renderer.start()         // subscribes to adapter.fetchConversation()
renderer.stop()          // unmounts, tears down observers
renderer.scrollToMessage(id)
renderer.setTheme(theme)
renderer.onMessageCountChange(cb: (count: number) => void): () => void
                         // fires whenever the post-patch message count changes;
                         // returns an unsubscribe fn. The shell uses this to
                         // persist the per-conversation count for threshold
                         // gating on subsequent visits.
```

`createRenderer` returns synchronously; `start()` is async (waits for first patch + font ready).

## Internal modules

### `core/blocks.ts`
Types only (the schema). No runtime code. Re-exports `BLOCK_SCHEMA_VERSION`.

### `core/adapter.ts`
The `Adapter` interface and `Capability` union (canonical definitions in `04-adapters.md`). Pure type module; no runtime. Lives in `core/` so that adapters depend *on the renderer-defined seam* rather than the renderer depending on `adapters/` — this preserves the boundary the architecture promises.

### `core/intents.ts`
The `Intent` union and `IntentResult` type (canonical definitions in `06-intents.md`). Pure types. Adapter `handleIntent` and renderer `onIntent` both reference these.

### `core/layout.ts` — pretext wrapper
- `measureText(runs, width, font, lineHeight) → { height, lines }`
- `measureCode(lines, width, font, lineHeight) → { height, longestLineWidth }`
- `measureOpaque(block, width) → { height }` (returns intrinsic, scaled if `reflow === 'fluid'`)
- `measureBlock(block, width, ctx)` dispatches by `block.kind`
- `measureMessage(message, width, ctx) → BlockMeasurement[]`

Wraps pretext's `prepare` / `prepareWithSegments` / `layout`. Pure functions; no DOM, no side effects beyond the height cache.

### `core/cache.ts` — height cache
- Key: `${messageId}:${blockIndex}:${width}` for blocks; rounds width to the nearest 8 px to avoid thrash on continuous resize.
- Eviction: LRU bounded at ~50k entries. Evict on message remove or message-block-replace.
- Invalidation hooks: `invalidate(messageId)`, `invalidate(messageId, blockIndex)`, `invalidateAll()` (theme/font change).

### `core/virtualizer.ts` — windowing
- Maintains `messages: Message[]` plus per-message cumulative top offsets.
- On scroll: binary-search the visible message range, expand by `overscanPx`.
- Mounts/unmounts message DOM nodes as the window moves.
- Within a mounted message, all blocks render (we virtualize at message granularity, not block, in v1 — simpler and good enough; revisit if a single message contains thousands of blocks, which is rare).

Why message-granularity windowing in v1:
- Stable scroll anchoring is much easier when whole messages enter/leave.
- Selection across visible messages works naturally.
- Profiling on real corpora suggests messages are the right unit; oversized messages are rare and handled with `content-visibility: auto` as a backstop.

### `core/mount.ts` — opaque host
- For each `OpaqueBlock`, mounts payload inside a sandboxed div.
- `payload.html`: rendered into `Trusted Types`-sanitized container (DOMPurify on input from adapter, even though adapter is trusted — defense in depth).
- `payload.image`: `<img loading="lazy" decoding="async">`.
- `payload.iframe`: sandboxed `<iframe>` for tool-call cards that need isolation.
- ResizeObserver tracks intrinsic changes for `reflow === 'fluid'` blocks (e.g., responsive embeds); reports back to the cache and triggers re-layout above this block.

### `core/runtime.ts` — scroll + lifecycle
- Owns the scroll container.
- IntersectionObserver for viewport tracking (low overhead vs scroll listeners).
- Handles font-loaded re-layout: `document.fonts.ready` once, then re-layout pass.
- Handles theme switch: re-paint only (no re-measure).
- Handles container width change via ResizeObserver on the container; re-layouts visible window first, defers off-screen until scrolled near.

### `core/theme.ts` — token color resolution
- Maps `TokenColor` names → hex per active theme.
- Two built-in themes for v1: `light` (matches host) and `dark` (matches host).
- Adapters can override by passing a theme override into `createRenderer({ theme })`.

### `core/intents.ts` — action emitter
- Block-level UI (copy button on code, link on opaque) calls `emit({ type, ... })`.
- The renderer collects these and forwards to `onIntent` (which by default delegates to `adapter.handleIntent`).
- Renderer never imports adapter code.

## Layout pass — algorithmic flow

1. **Receive patch** from `adapter.fetchConversation()`.
2. **Apply patch** to in-memory message list. For `add`/`replace`/`replaceBlocks`, invalidate cache for affected messages.
3. **Measure on demand**: virtualizer asks for the height of message `i` at the current container width. If cached → return; else `measureMessage(message, width)`, store, return.
4. **Update cumulative offset table** for messages whose height changed.
5. **Reconcile mounted window**: if the visible range changed, mount new messages, unmount old.
6. **Paint**: pretext-laid lines render as `<span>` lines inside a `<pre class="ptx-text">`-style container; opaque blocks render via `mount.ts`.

Steps 3–5 are O(visible ± overscan), not O(total messages). Step 4 is O(messages_with_changed_height), typically 1–2 (the streaming message and any preceding patched message).

## Streaming layout

Streaming hits one block per tick. Two cases:

- **Text append** (`{ op: 'append' }`): pretext re-layouts that single block. If the block grew by N new lines, the message height grows by `N * lineHeight`. Cumulative offset table updates O(messages-after-the-streaming-one). This is fine because the streaming message is usually last.
- **Code line append** (`{ op: 'appendLine' }`): same, but no re-tokenization until the fence closes (replace patch arrives).

Throttle: the renderer batches patches per animation frame. Multiple `append` patches within one frame fold into a single layout pass.

## Selection and copy

- Pretext lines are real `<span>` text nodes — native browser selection works.
- Copy from a code block returns `CodeBlock.source` verbatim (override `copy` event when selection lies entirely within a code block).
- Copy across multiple blocks uses the natural DOM concatenation; opaque blocks contribute `OpaqueBlock.copyText` if set, else their visible text content.

## Search (Cmd+F)

Browser find only sees mounted content. v1 mitigation:
- "Find in conversation" button (in renderer chrome) opens a custom search bar; matches scroll mounted blocks into view.
- Browser-native Cmd+F still works for the visible window — acceptable for v1.
- v2: temporarily mount entire conversation into `display: none` shadow tree on Cmd+F press. Deferred.

## Accessibility

- The mounted window is keyboard-navigable (arrow keys move focus between messages).
- ARIA: each message gets `role="article"` with author label.
- "Render full transcript" toggle in renderer chrome forces all messages mounted (slow but accessible) for screen readers and printing.
