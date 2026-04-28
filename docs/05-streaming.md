# Streaming

Streaming is the hardest seam between adapter and renderer. The protocol below keeps the renderer dumb (no markdown parsing in the hot path) while keeping latency low.

## Principle

**The adapter parses; the renderer applies.** The renderer never sees raw markdown or SSE bytes. It receives only `MessagePatch`es shaped exactly like steady-state edits, so streaming and history use the same code path.

## Patch types relevant to streaming

```ts
{ op: 'add';         message: Message }                                   // turn starts
{ op: 'append';      messageId; blockIndex; run: Run }                    // text grows
{ op: 'appendLine';  messageId; blockIndex; text: string; line: Run[] }   // code line lands
{ op: 'replace';     messageId; blockIndex; block: Block }                // create-or-replace a block at index
{ op: 'patchMeta';   messageId; meta: Partial<Message['meta']> }          // shallow-merge into Message.meta
{ op: 'finalize';    messageId }                                          // turn ends
```

`replace` is **create-or-replace**: if no block exists at `blockIndex` yet, it creates one. The contract requires `blockIndex === blocks.length` for creation (no holes) — see invariants in `02-block-schema.md`. Subsequent `append` / `appendLine` patches against the same index are valid only after the block has been created via `replace`.

`patchMeta` is the only patch that mutates `Message.meta`. Adapters use it for stream signals the renderer cares about (`truncated`, `model`, etc.) without resending blocks.

A typical assistant turn in flight:

```
add        { id: 'm_42', role: 'assistant', status: 'streaming', blocks: [] }
replace    blockIndex 0 → text block, runs: [{ text: 'Sure, here' }]   (creates block 0)
append     blockIndex 0, run: { text: ' is the' }
append     blockIndex 0, run: { text: ' code:' }
replace    blockIndex 1 → code block, language: '', source: '', lines: []   (creates block 1)
appendLine blockIndex 1, text: 'def hello():',     line: [{ text: 'def hello():', mono: true }]
appendLine blockIndex 1, text: '    print("hi")',  line: [{ text: '    print("hi")', mono: true }]
replace    blockIndex 1 → code block re-tokenized with language: 'python'
                          (source unchanged; lines now carry TokenColor runs)
replace    blockIndex 2 → text block, runs: [{ text: '' }]   (creates block 2)
append     blockIndex 2, run: { text: '\nThat should work.' }
finalize   messageId: 'm_42'
```

## Why two append flavors

- **`append`** (text): pretext re-flows that one text block. Cheap. The text inside often crosses word/line boundaries mid-stream, so we want pretext's wrapping to handle it — appending a Run is the right granularity.
- **`appendLine`** (code): code is line-oriented and monospace; lines never wrap inside a fence. Appending whole lines avoids re-measuring partial lines and lets the renderer extend the code block height by exactly `lineHeight` per patch. The patch carries both `text` (raw, appended to `CodeBlock.source`) and `line` (the tokenized `Run[]` appended to `CodeBlock.lines`); the renderer applies both atomically so the source ↔ lines invariant always holds.

Mixing the two simplifies adapter parsing: stream the text bytes through a small markdown state machine; emit `append` for prose, switch to line-buffered emission inside fences (`appendLine` per `\n`).

## Re-tokenization timing

While a code fence is open, the adapter cannot run a syntax highlighter — the source isn't complete and language may not be declared. Strategy:

1. **During** the fence: emit `appendLine` with `[{ text: line, mono: true }]` (no colors). Renderer shows uncolored monospace.
2. **At fence close**: adapter calls `services/highlight.ts` with full source + detected language. Emits a `replace` with the fully tokenized `CodeBlock`.

The replace doesn't change the line count, so the height delta is zero and there's no scroll jump. Only the color runs change → repaint, no re-layout.

## Latency budget

For perceived smoothness, streaming text must paint within one frame of arrival.

- Adapter parses one SSE chunk → emits one or more patches.
- Renderer batches all patches that arrive within a `requestAnimationFrame` tick.
- One layout pass per frame, regardless of patch count.

Worst case in a frame: many `append` patches to the same block. Pretext re-layout of one text block is O(lines in block). For a 2000-line message (extreme), that's still well under 1 ms on a modern laptop.

## Backpressure

If patches arrive faster than the renderer can apply them (e.g., dumping a 1 MB message at once), the runtime drops to a coarser strategy:

- Coalesce all `append` patches for the same `(messageId, blockIndex)` within the frame into a single Run with concatenated text.
- If the queue depth exceeds N (default 256), defer non-visible message updates until the streaming message finalizes.

Visible window is always prioritized; off-screen patches can fall behind without user-visible cost.

## Edits and regenerations

- **User edits a previous message** → adapter emits `replaceBlocks` for that message. Cache invalidates for that message only; offsets for messages below it shift if height changed.
- **User clicks regenerate** → adapter emits:
  - `remove` for the old assistant message (if site discards it), or
  - `add` for the new attempt with `parentId` pointing at the user message (if site keeps a branch tree).
- **Branch switching** in branched conversations → adapter emits a series of `remove` + `add` patches to align the visible chain.

## Streaming the user message

User messages are fully composed before sending — they don't stream. When the user submits, adapter emits a single `add` with the complete message. No special path.

## Failure modes during stream

- **Network drops mid-stream** → adapter emits `replace` of the streaming block with whatever was buffered, then `patchMeta { truncated: true }`, then `finalize`. Renderer can show a "(stream interrupted)" affordance based on `meta.truncated`.
- **Site sends a duplicate `add`** → renderer dedupes by `id`; the second `add` becomes a `replaceBlocks`.
- **`append` references a block that doesn't exist** → renderer logs and ignores in prod, throws in dev. Indicates an adapter bug — adapters must create the block via `replace` first.

## Test strategy

- **Replay tests**: capture a real SSE session, feed bytes through the adapter, assert the patch sequence matches a golden file.
- **Patch fuzzer**: random valid patch streams driven into the renderer; assert the final state matches a fully-loaded equivalent.
- **Frame-budget tests**: synthetic 100-patches-per-frame load; assert no frame exceeds 16 ms in a headless run.
