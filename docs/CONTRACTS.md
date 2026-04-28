# Cross-cutting contracts (single source of truth)

The other docs describe behavior. This file is the index of the **shared types** that flow across the seam between layers. If a doc references one of these names, it must match the canonical definition listed here. When a contract changes, change it in the canonical file first, then update the index entry below.

| Contract | Canonical doc | Type module | Used by |
|---|---|---|---|
| `Block`, `Run`, `TokenColor`, `Message`, `Role` | [`02-block-schema.md`](02-block-schema.md) | `core/blocks.ts` | renderer, adapters, `services/markdown.ts`, `services/highlight.ts` |
| `MessagePatch` (the patch union) | [`02-block-schema.md`](02-block-schema.md) | `core/blocks.ts` | renderer, adapters; streaming detail in [`05-streaming.md`](05-streaming.md) |
| `Adapter` (the adapter interface) | [`04-adapters.md`](04-adapters.md) | `core/adapter.ts` | implemented by per-site modules under `adapters/`; consumed by the renderer and shell. Wiring rules in "Module ownership" below. |
| `Capability` (the capability union) | [`04-adapters.md`](04-adapters.md) | `core/adapter.ts` | renderer (UI gating in [`03-renderer.md`](03-renderer.md)), intents ([`06-intents.md`](06-intents.md)) |
| `Intent`, `IntentResult` | [`06-intents.md`](06-intents.md) | `core/intents.ts` | renderer, adapters |
| `BLOCK_SCHEMA_VERSION` | [`03-renderer.md`](03-renderer.md) | `core/blocks.ts` | adapters declare compatibility via `Adapter.schemaVersions` |

All seam types live in `core/` so the dependency direction is `adapters/ → core/`, never the reverse. The renderer depends on the *interface* it defines, not on any concrete adapter.

## Rules

1. **One canonical file per contract.** Other docs may quote a contract for context but must not silently extend it. New fields, ops, or capability names land in the canonical file first.
2. **Cross-references must round-trip.** If `08-extension-shell.md` calls `adapter.findMessageList(...)`, that method must exist on the `Adapter` interface in `04-adapters.md`. If `06-intents.md` gates an intent on `'devTools'`, that name must be in the `Capability` union in `04-adapters.md`.
3. **Schema version bumps are renderer-coordinated.** Any breaking change to `Block`, `MessagePatch`, or the `Adapter` interface bumps `BLOCK_SCHEMA_VERSION`. The shell refuses to wire an adapter whose `schemaVersions` doesn't include the renderer's current version.

## Layering invariants (from [`01-architecture.md`](01-architecture.md))

- The **seam modules** in `core/` are types-only, the single source of truth for cross-layer contracts:
  - `core/blocks.ts` — `Block`, `Run`, `Message`, `MessagePatch`, `BLOCK_SCHEMA_VERSION`.
  - `core/adapter.ts` — `Adapter`, `Capability`.
  - `core/intents.ts` — `Intent`, `IntentResult`.
  Each is a pure type module (the schema-version constant is the only runtime export).
- Renderer, adapters, and services may all import from the seam modules.
- Renderer imports nothing from `adapters/` or `services/` (the seam types live on the renderer side specifically so the renderer never has to reach across).
- Adapters import from the seam modules only (no `createRenderer`, no renderer internals, no other `core/` runtime).
- Services import from the seam modules only (typically just `core/blocks.ts`); no `adapters/` imports.
- The shell is the only module that imports both `createRenderer` and a concrete `Adapter` implementation and wires them together.

## Module ownership (who imports `createRenderer`)

| Module | Imports `createRenderer`? | Imports an `Adapter` impl? | Notes |
|---|---|---|---|
| `core/` (renderer)            | n/a (defines it)              | no | Receives `Adapter` *as a parameter* via `createRenderer({ adapter })`; never imports a concrete adapter. |
| `adapters/<site>/`            | **no** (forbidden by ESLint)  | n/a (defines them) | Adapter modules expose an `Adapter` value; they do not know `createRenderer` exists. |
| `services/`                   | **no**                        | **no** | Leaf utilities. Only `core/blocks.ts` from `core/`. |
| `shell/content.ts`            | **yes**                       | **yes** | The single wiring point. Picks an adapter via `matches()`, calls `createRenderer({ adapter })`. |

If an `import` of `createRenderer` ever appears outside `shell/`, the ESLint boundary rule fails CI. This is the seam the architecture is built to enforce.

## Invariants & sequencing

The type tables above pin down *shapes*. These rules pin down *behavior across patches and lifecycle steps*. They are the invariants future doc edits must round-trip against; if a doc change would violate one of these, the rule must be updated here in the same change.

### Block / message invariants
- `Message.id` is unique within a session.
- `CodeBlock`: when `source === ''`, `lines.length === 0` (canonical empty); otherwise `lines.length === source.split('\n').length`. Holds at every observable state, not just finalize. Streaming patches must keep `source` and `lines` in lockstep — the renderer applies `appendLine`'s `text` and `line` atomically.
- `OpaqueBlock.intrinsicWidth > 0 && intrinsicHeight > 0`.
- `Run.color` is set only inside `CodeBlock.lines`; ignored elsewhere.

### Patch protocol invariants
- `replace` is the only patch that creates a block. It may create only at `blockIndex === blocks.length` (no holes).
- `append` and `appendLine` against a non-existent block are an adapter bug — the renderer logs and ignores in prod, throws in dev.
- `append` only targets `text` blocks; `appendLine` only targets `code` blocks.
- `appendLine` carries both `text` (raw) and `line` (tokenized). The renderer applies both atomically; it is illegal to send one without the other.
- `replace` does not change `kind` from `text` / `code` to `opaque` mid-stream (would invalidate height assumptions during scroll).
- `patchMeta` is the only patch that mutates `Message.meta`. Stream signals like `truncated` flow through `patchMeta`, not `replace`.
- After `finalize`, no further patches reference that `messageId` except `remove`, `replaceBlocks` (user edit), or another `add` with the same id (adapter bug → renderer treats as `replaceBlocks`).

### Shell sequencing invariants
- The shell decides whether to activate the renderer **before** mutating the native UI. No `findMessageList(...).dataset.cpHidden = '1'` until the threshold decision is made.
- The shell is the only module that imports both `createRenderer` and an `Adapter` instance. Adapters never name `createRenderer`.
- On SPA navigation, the shell must un-hide the native list and unmount before re-running the entry function — leaving stale `cpHidden` state across conversations is a bug.
- **Hidden ⇒ teardown.** While any DOM mutation (`cpHidden`, mounted viewport) or adapter resource (interceptor, observer) is live, `active.teardown` must be registered. Therefore the shell registers `active.teardown` *before* awaiting `renderer.start()`, and any failure path (including `start()` throwing) calls it.
- **Teardown disposes the adapter.** Every `active.teardown` calls `adapter.dispose()`. Skipping it leaks listeners across SPA navigation and produces stale patches against the next conversation.

### Capability ↔ intent round-trip
- Every `Intent` type whose docs say "gated by capability X" requires X in the `Capability` union.
- Every `Capability` value must either gate at least one `Intent`, an Adapter behavior (e.g. `'streaming'`), or a renderer UI affordance. Unused capability values are dead code.

### Async lifecycle invariants
- **Adapter input is untrusted.** Bytes arriving on the page-hook port (or any MAIN ↔ ISOLATED channel) MUST be schema-validated by the adapter before being turned into `MessagePatch`es. MV3 does not provide a fully private cross-world channel: (a) other extensions' content scripts can observe the bootstrap and obtain a port reference; (b) page scripts can read the bootstrap secret from the DOM and forge a `cp:bootstrap` ahead of the real page-hook. The "secret" is collision-avoidance, not authentication. Adapters defend by treating the byte stream as adversarial input — bad shape → parse error → no patches emitted, never silent corruption. See `08-extension-shell.md#honest-threat-model` for the full discussion.
- **Stale session rule.** Any `await` inside the shell's `main()` (or any equivalent re-enterable lifecycle function) is a yield point at which a newer invocation may have superseded this one. Before mutating shared state (`active`, the DOM mount, `cpHidden`) on resume, the function must check that its per-invocation token is still current; if not, it must clean up only what it itself created (typically `adapter.dispose()` and any partial mount it inserted) and exit without touching `active`.
- **Late-rejection rule.** A promise rejection from a side-effecting call (`renderer.start()`, network fetches) may arrive after the session has been superseded. The catch handler must check session identity before invoking `active.teardown()`; otherwise it can tear down the *current* session by accident. Use a captured `session` reference and compare `active?.id === session.id`.
- **IPC transferability rule.** Anything sent across the MAIN ↔ ISOLATED world boundary must be a structured-cloneable value. `ReadableStream` is not reliably cloneable today — pump it in the source world and forward chunks (`Uint8Array`, `string`, plain objects) instead. Tag each stream with a `streamId` so the receiver can demultiplex concurrent streams.
- **Private channel rule.** All MAIN ↔ ISOLATED traffic after handshake flows over a `MessageChannel`. `window.postMessage` is broadcast to every script in the page and unsafe for conversation data; using it for stream chunks would let any other MAIN-world script forge adapter input.
- **Order-independent bootstrap.** MV3 does not guarantee MAIN/ISOLATED `document_start` script ordering. The handshake must work in either order: ISOLATED installs its bootstrap listener *before* writing the readiness attribute (`document.documentElement.dataset.cpIsolated = secret`); MAIN tries the attribute synchronously on load and falls back to a `MutationObserver` if absent. A one-shot bootstrap in either direction is a bug.
- **No idle teed branches.** `ReadableStream.tee()` buffers any branch that isn't actively being read while the other branch is consumed. Therefore the page-hook must start an active reader on its tee branch *immediately* upon interception, even if the destination port is not yet bound. Pre-handshake streams pump into a bounded in-memory queue with a handshake-timeout; on cap or timeout the reader is cancelled to release the tee buffer.
- **Origin-checked bootstrap receiver.** ISOLATED's single `window.addEventListener('message', ...)` listener (the bootstrap claim) validates `event.origin === location.origin`, `event.source === window`, the per-page-load secret, and `event.ports.length === 1` before claiming `event.ports[0]`. Once the channel is established no further `postMessage` listeners are needed — port traffic does not flow through the global `message` event.

## When to update this file

- A new patch op is added → update `MessagePatch` row **and** add its invariants to "Patch protocol invariants" above.
- A new capability flag is added → update `Capability` row **and** add its gated intent / behavior to "Capability ↔ intent round-trip".
- A new method is added to the adapter interface → update `Adapter` row.
- A doc starts referencing a cross-cutting type that isn't listed → add a row before merging.
- A doc changes shell startup ordering, navigation, or any cross-doc behavior → update "Shell sequencing invariants" in the same change.
