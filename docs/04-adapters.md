# Layer 2: Adapters (`src/adapters/`)

A site adapter knows everything about one chat product (ChatGPT, Grok, …) and nothing about the renderer. Its job: produce `MessagePatch`es and execute `Intent`s.

## Adapter interface

The `Adapter` interface and `Capability` union below are the **canonical declarations**, but the **type module** lives in `core/adapter.ts` so adapters depend on a renderer-side seam rather than the renderer depending on `adapters/`. Adapter modules import the type:

```ts
import type { Adapter, Capability } from '../../core/adapter'
```

```ts
export interface Adapter {
  /** Stable name; used for logging and per-site settings. */
  readonly name: string

  /** Schema versions this adapter is compatible with. */
  readonly schemaVersions: readonly number[]

  /** What this adapter can do; renderer uses this to gate UI. */
  readonly capabilities: ReadonlySet<Capability>

  /** True iff this adapter should activate on the current page. */
  matches(location: Location, document: Document): boolean

  /** Returns an async iterable of patches. The renderer drains it. */
  fetchConversation(signal: AbortSignal): AsyncIterable<MessagePatch>

  /** Execute a user-triggered intent. Renderer awaits success/failure. */
  handleIntent(intent: Intent): Promise<IntentResult>

  /**
   * Locate the site's existing message-list element inside the chat container
   * the shell has waited for. The shell hides this element and mounts the
   * pretext viewport in its slot. DOM-shaped because every supported site
   * structures its conversation page differently.
   */
  findMessageList(container: HTMLElement): HTMLElement

  /**
   * Extract the conversation id from a URL during SPA navigation. Used by
   * the shell to render a stable placeholder while swapping renderers, so
   * there's no flash of empty state. Return `null` if the URL isn't a
   * conversation route (landing page, settings, etc.).
   */
  parseConversationIdFromUrl(url: URL): string | null

  /** Tear down listeners, network interceptors, observers. */
  dispose(): void
}

export type Capability =
  | 'streaming'         // emits append/appendLine patches
  | 'edit'              // can submit a user-message edit
  | 'regenerate'        // can request a new assistant response
  | 'feedback'          // can submit thumbs up/down
  | 'copy'              // can copy a message via site mechanism (else fallback)
  | 'branchHistory'     // exposes parentId for regenerated turns
  | 'reportBug'         // can attach diagnostics and open a feedback form
  | 'devTools'          // privileged debug intents (e.g. jumpToOriginal)
  | 'openLinkOverride'  // adapter overrides default window.open for links
  | 'openImageOverride' // adapter overrides default window.open for images

// `IntentResult` is defined canonically in `06-intents.md` (it is part of the
// intent protocol, not the adapter shape). Adapters import it from the same
// module that exports `Intent`.
```

## Two strategies for getting data

### Strategy A: Network interception (preferred)

Hook `fetch` / `XMLHttpRequest` and listen for the conversation endpoint. Parse the same JSON the site's React app consumes.

Pros:
- **Schema is more stable than DOM.** Backend APIs change less often than markup.
- Streaming arrives as SSE/JSON-lines; trivially turned into `append` patches.
- Conversation history loads via paginated calls; we can ask for all of it explicitly.

Cons:
- Endpoints occasionally rotate or move under different domains (ChatGPT has done this).
- Auth headers and CSRF tokens already attached by the page — we just observe.

Implementation: monkey-patch `window.fetch` early in the content script (before page scripts run, via `run_at: document_start` and a `world: MAIN` injection). Filter responses by URL pattern, branch on shape.

### Strategy B: DOM scraping (fallback)

`MutationObserver` on the original message list. Read text + structure. Used when network interception is impractical (e.g., sites that render fully server-side or use unguessable URLs).

Pros:
- Works against any site eventually.
- Doesn't depend on backend.

Cons:
- Brittle to markup changes.
- Loses information the DOM doesn't carry (e.g., raw markdown source — must reverse-engineer from rendered HTML).
- Streaming detection relies on observing partial DOM updates; messy.

For ChatGPT and Grok in v1: **Strategy A**. Strategy B is the v3 generic fallback.

## Adapter pipeline (per site)

```
network/DOM  →  raw payload (JSON or HTML fragment)
                 │
                 ▼
            normalize        (site-specific shape → canonical message shape)
                 │
                 ▼
         markdown.parse      (services/markdown.ts → md AST)
                 │
                 ▼
            blockify         (md AST → Block[]; calls highlight/katex/image services)
                 │
                 ▼
        emit MessagePatch    (op: 'add' | 'append' | 'replace' | ...)
```

The first three stages live in the per-site folder; the fourth is the contract output.

## ChatGPT adapter notes

- Conversation endpoint pattern (subject to change): `/backend-api/conversation/{id}` for history, SSE on `/backend-api/conversation` for new turns.
- Message shape uses a "mapping" tree — turns can have multiple children when regenerated. Walk the active branch via `current_node`.
- Code language detection from fence info-string; fallback to highlighter's language guess.
- Math fences: `\(...\)`, `\[...\]`, `$$...$$`. Send as `OpaqueBlock` via `services/katex.ts`.
- Tool calls (browsing, code interpreter, image gen) arrive as message parts with custom types — render as `OpaqueBlock` with an `iframe` payload that mirrors the original card markup, OR as a styled HTML block. v1: HTML mirror; v2: live cards.
- Edit, regenerate, feedback all hit known endpoints — proxy via the page's existing auth context.

## Grok adapter notes

- Different conversation API; SSE-based streaming similar in spirit.
- Less mature feature set — no tool-call cards in v1; treat any unrecognized part as an `OpaqueBlock` containing the raw site rendering.
- Threading model is linear (no branched regenerations as of v1) — `parentId` always = previous message id.

## Intent handling

```ts
// User clicks "copy" on a code block in the renderer.
// Renderer emits:
{ type: 'copy', messageId: 'm_42', blockIndex: 3 }

// Adapter:
async handleIntent(intent: Intent): Promise<IntentResult> {
  switch (intent.type) {
    case 'copy': {
      // Find the original site button or fall back to clipboard API.
      const block = this.cache.get(intent.messageId)?.blocks[intent.blockIndex]
      if (block?.kind === 'code') {
        await navigator.clipboard.writeText(block.source)
        return { ok: true }
      }
      // ... full message copy path
    }
    case 'edit':       return this.proxyEdit(intent.messageId)
    case 'regenerate': return this.proxyRegenerate(intent.messageId)
    case 'feedback':   return this.proxyFeedback(intent)
    default:           return { ok: false, reason: 'unsupported' }
  }
}
```

Where possible, `handleIntent` triggers the **site's own UI flow** (focuses the original edit textarea, clicks the original regenerate button) rather than re-implementing the action against the API. This keeps us out of authentication and rate-limit territory.

## Adapter testing

- **Snapshot tests** on captured raw payloads (committed under `test/fixtures/chatgpt/payloads/`). Adapter must produce a stable `Message[]` from each fixture.
- **Schema-conformance test** runs every fixture and asserts the renderer's invariants (see `02-block-schema.md`) hold on every emitted block.
- **Capability matrix test** ensures `capabilities` accurately reflects which intents `handleIntent` actually supports.

## Adapter capability flags drive renderer UI

- `capabilities.has('edit')` → show edit button on user messages.
- `capabilities.has('regenerate')` → show regenerate button on assistant messages.
- `capabilities.has('feedback')` → show thumbs.
- Missing capability → button hidden. No empty/broken affordances.
