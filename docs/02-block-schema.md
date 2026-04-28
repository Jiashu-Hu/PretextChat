# Block Schema (the contract)

This is the only data shape that crosses the adapter ↔ renderer seam. Both sides depend on this file; nothing else couples them.

## Design principles

1. **Renderer-friendly, not parser-friendly.** Adapters do the work of normalizing site-specific quirks into a shape the renderer can lay out without conditionals.
2. **No site-specific fields.** No `chatgptMessageId`, no `grokTurn`. Site-specific state stays inside the adapter.
3. **Pre-tokenized, pre-measured.** Code arrives already tokenized; opaque blocks arrive with intrinsic dimensions. The renderer never calls Shiki, KaTeX, or `Image()`.
4. **Stable IDs.** Every Message has a stable `id` for cache lookup, edit invalidation, and scroll restoration.
5. **Token-color names, not hex.** Theme resolution lives in the renderer.

## Types

```ts
// ── runs ────────────────────────────────────────────────────────────────────
//  An inline span of styled text. Used inside text and code blocks.

export type Run = {
  text: string
  weight?: 'normal' | 'bold'
  italic?: boolean
  mono?: boolean
  underline?: boolean
  strike?: boolean
  href?: string                 // makes it a link
  color?: TokenColor            // for code; resolved by renderer per theme
  title?: string                // hover title (e.g. for footnote markers)
}

// Semantic token names. Renderer maps name → hex per active theme.
// Keep this list small and stable; adding a name is a renderer change.
export type TokenColor =
  | 'fg' | 'muted'
  | 'keyword' | 'string' | 'number' | 'comment'
  | 'function' | 'type' | 'variable' | 'constant'
  | 'operator' | 'punctuation' | 'tag' | 'attr'
  | 'added' | 'removed' | 'changed'

// ── blocks ──────────────────────────────────────────────────────────────────

export type TextBlock = {
  kind: 'text'
  runs: Run[]
  indent?: number               // em units; for nested list/quote rendering
}

export type HeadingBlock = {
  kind: 'heading'
  level: 1 | 2 | 3 | 4 | 5 | 6
  runs: Run[]
  anchorId?: string             // for in-page anchors if present
}

export type QuoteBlock = {
  kind: 'quote'
  children: Block[]             // nested blocks rendered with quote bar
}

export type ListBlock = {
  kind: 'list'
  ordered: boolean
  start?: number                // ordered list start index
  items: Block[][]              // each item is its own block sequence
}

export type CodeBlock = {
  kind: 'code'
  language: string              // adapter's best guess; '' if unknown
  source: string                // the original raw source (for copy)
  lines: Run[][]                // pre-tokenized; one Run[] per source line
  // Note: width/height are computed by the renderer; not stored here.
}

export type OpaqueBlock = {
  kind: 'opaque'
  // What to mount. The renderer will host this in a sandboxed container.
  payload:
    | { type: 'html';  html: string }
    | { type: 'image'; url: string; alt?: string }
    | { type: 'iframe'; src: string; sandbox?: string }
  intrinsicWidth: number        // CSS px at intrinsic density
  intrinsicHeight: number       // CSS px
  reflow: 'fixed' | 'fluid'     // 'fluid' = re-measure on width change
  copyText?: string             // what Cmd+C should yield (e.g. LaTeX source)
  // Stable identity for cache reuse across re-renders.
  cacheKey: string              // e.g. sha1(latexSource) or url
}

export type Block =
  | TextBlock
  | HeadingBlock
  | QuoteBlock
  | ListBlock
  | CodeBlock
  | OpaqueBlock

// ── messages ────────────────────────────────────────────────────────────────

export type Role = 'user' | 'assistant' | 'system' | 'tool'

export type Message = {
  id: string                    // stable, adapter-assigned
  role: Role
  blocks: Block[]
  status: 'final' | 'streaming'
  createdAt?: number            // unix ms
  parentId?: string             // for branched conversations (regenerations)
  meta?: {
    model?: string              // e.g. 'gpt-5', 'grok-3'
    truncated?: boolean         // adapter signals incomplete data
    [key: string]: unknown      // adapter-private; renderer ignores
  }
}
```

## Patches (streaming and edits)

Streaming and edits do not re-send the whole message. Adapters emit incremental patches:

```ts
export type MessagePatch =
  // Whole new message arrives (initial load or new turn)
  | { op: 'add';     message: Message }

  // Append a Run to the tail of an existing text block (streaming prose).
  // For code, use `appendLine`. Targeting any non-text block is an adapter bug.
  | { op: 'append';  messageId: string; blockIndex: number; run: Run }

  // Append a fully-formed line to a code block (streaming code, line-buffered).
  // `text` is the raw line. The renderer appends it to CodeBlock.source as:
  //   source === '' ? text : source + '\n' + text
  // and pushes `line` onto CodeBlock.lines, atomically — so the source ↔ lines
  // invariant in §Invariants holds after every patch. `line` is the same
  // content already tokenized into runs (during a fence with unknown language,
  // this is typically a single mono Run).
  | { op: 'appendLine'; messageId: string; blockIndex: number; text: string; line: Run[] }

  // Create-or-replace a block at blockIndex. For creation, blockIndex must
  // equal blocks.length (no holes). Used both for new blocks during streaming
  // and to swap an existing block (re-tokenize a code fence on close, swap a
  // placeholder image for the loaded one).
  | { op: 'replace'; messageId: string; blockIndex: number; block: Block }

  // Replace the entire block list (e.g., user edited their message)
  | { op: 'replaceBlocks'; messageId: string; blocks: Block[] }

  // Shallow-merge into Message.meta. The only patch that mutates meta;
  // adapters use it for stream signals like { truncated: true }.
  | { op: 'patchMeta'; messageId: string; meta: Partial<NonNullable<Message['meta']>> }

  // Mark message complete; renderer can flip status: 'streaming' → 'final'
  | { op: 'finalize'; messageId: string }

  // Remove a message (e.g., regenerate destroys the previous attempt)
  | { op: 'remove'; messageId: string }
```

The adapter exposes patches as an `AsyncIterable<MessagePatch>`; the renderer drains it.

## Invariants (enforced in dev builds)

1. `Message.id` is unique within a session.
2. `Block.kind === 'code'` ⇒
   - if `source === ''` then `lines.length === 0` (the canonical "empty / not-yet-streamed" code block);
   - otherwise `lines.length === source.split('\n').length`.
   This holds at every observable state, including mid-stream after each `appendLine` — the renderer applies `text` and `line` atomically.
3. `OpaqueBlock.intrinsicWidth > 0 && intrinsicHeight > 0`.
4. `MessagePatch.op === 'append'` only targets a `text` block; use `appendLine` for code.
5. `replace` does not change `kind` from `text`/`code` to `opaque` mid-stream (would invalidate height assumptions during scroll).
6. `Run.color` is set only inside `CodeBlock.lines`; ignored elsewhere.
7. `replace` may create a block only when `blockIndex === blocks.length` (no holes); `append` / `appendLine` against a missing block is an adapter bug.

In dev builds, the renderer asserts these on input and throws. In prod, it logs and skips.

## What is intentionally NOT in the schema

- **Per-message DOM handles.** Renderer owns the DOM; adapter never touches it.
- **Highlighter language plugins, KaTeX macros.** These are adapter concerns; result lands as `Run[][]` or `OpaqueBlock`.
- **Selection state, scroll position.** Owned by the renderer's runtime.
- **Adapter callbacks.** Use the intent protocol (`06-intents.md`), not function references inside blocks.
- **Markdown source string.** The adapter parses once; the renderer never sees raw markdown. (Exception: `CodeBlock.source` is kept for copy fidelity.)

## Versioning

The schema is versioned via `BLOCK_SCHEMA_VERSION` constant exported from `core/blocks.ts`. Breaking changes bump the major version. Adapters declare the schema versions they support; the shell refuses to wire incompatible pairs.
