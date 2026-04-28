# Intents (action protocol)

The renderer never calls adapter methods directly for user actions. It emits **intents** — declarative requests describing what the user wants to do — and the adapter handles them. This keeps the renderer site-agnostic and makes the action surface easy to mock in tests.

## Intent types

```ts
export type Intent =
  | { type: 'copy';           messageId: string; blockIndex?: number }
  | { type: 'copyMessage';    messageId: string; format: 'plain' | 'markdown' }
  | { type: 'edit';           messageId: string }
  | { type: 'regenerate';     messageId: string }
  | { type: 'feedback';       messageId: string; value: 'up' | 'down'; comment?: string }
  | { type: 'openLink';       href: string; messageId: string }
  | { type: 'openImage';      url: string; messageId: string }
  | { type: 'jumpToOriginal'; messageId: string }   // scroll site's hidden DOM into view
  | { type: 'reportBug';      messageId: string; kind: 'parse' | 'layout' | 'other' }

export type IntentResult =
  | { ok: true }
  | { ok: false; reason: 'unsupported' | 'failed' | 'denied'; detail?: string }
```

`IntentResult` is the canonical return type for `Adapter.handleIntent` and the renderer's `onIntent` callback. Both layers import it from this module.

## Why declarative, not callbacks

```ts
// ❌ Don't do this — couples block data to adapter
type CodeBlock = {
  ...
  onCopy: () => void  // adapter callback baked into block data
}

// ✅ Do this — block stays pure data; adapter handles intent
renderer.onIntent = (intent) => adapter.handleIntent(intent)
```

Benefits:
- Blocks remain serializable. We can dump and replay them in tests.
- Renderer can log/audit/queue intents without knowing what they do.
- Multiple adapters (e.g., a future "offline export" adapter) implement the same intent shape.
- The renderer can synthesize intents from keyboard shortcuts, context menus, or programmatic calls — same path as button clicks.

## Capability gating

Intents the adapter doesn't support are gated at UI render time, not at click time:

```ts
// Renderer chrome:
if (adapter.capabilities.has('regenerate')) {
  showRegenerateButton(message)
}
```

If a user-triggered intent reaches an adapter that can't handle it, the adapter returns:

```ts
{ ok: false, reason: 'unsupported' }
```

The renderer shows a small toast: *"This action isn't available on this site."* This is a defensive path — capability gating should normally prevent the intent from being constructable.

## Routing rules

1. **Renderer-handled defaults first.** `openLink` and `openImage` are universal browser actions, not site-specific. The renderer handles them itself — `openLink` calls `window.open(href, '_blank', 'noopener,noreferrer')`; `openImage` opens `url` the same way (or in a lightbox if one is enabled). These intents never round-trip through the adapter unless the adapter explicitly opts in (see below). They have no `Capability` flag because every site supports them by definition.
2. **All other intents → `onIntent` callback** (default = `adapter.handleIntent`).
3. The callback is awaited; the renderer can show a pending state on the originating UI element.
4. On `{ ok: true }` → success state (e.g., "Copied!" toast for 1.5 s).
5. On `{ ok: false }` → error state with reason; never silently swallowed.

### Adapter override of renderer-handled intents

An adapter that wants to customize `openLink` or `openImage` (e.g., site-internal navigation that should stay in the SPA, or a richer image viewer) implements `handleIntent` for them and exposes a capability:

- `'openLinkOverride'` — adapter takes responsibility for `openLink`.
- `'openImageOverride'` — adapter takes responsibility for `openImage`.

When the renderer sees these capabilities, it dispatches to the adapter instead of calling `window.open`. Without them, the renderer never bothers the adapter with these intents, so default adapters do not need to implement them at all.

Intents are **fire-and-await**, not subscribed. If the action causes downstream changes (e.g., regenerate produces a new assistant message), those arrive through the patch stream — not via the intent's return value.

## Side effects from intents

Most intents trigger conversation changes that flow back through `fetchConversation()` as patches:

```
user clicks Edit
  → renderer emits { type: 'edit', messageId: 'm_42' }
  → adapter.handleIntent focuses site's edit textarea, returns { ok: true }
  → user types and submits in the original UI
  → site issues a new turn over the network
  → adapter intercepts, emits 'replaceBlocks' (edited user msg) + 'add' (new assistant turn)
  → renderer applies patches; UI updates
```

The renderer does not need to know that an edit eventually produces new patches — it just keeps draining the iterable.

## Privileged intents

Some intents perform actions that go beyond the page's normal flow:

- **`reportBug`** — opens a feedback form (in a popup or new tab) with diagnostic info attached. Adapter writes nothing remotely without user confirmation.
- **`jumpToOriginal`** — scrolls the site's hidden original DOM into view (debug feature; only when `debug: true`).

These are flagged in capability set as `'reportBug'`, `'devTools'`.

## Synthesized intents from keyboard

The renderer maps a small set of keys to intents inside the focused message:

| Key             | Intent                                     |
|-----------------|--------------------------------------------|
| `c`             | `copyMessage` (markdown)                   |
| `e`             | `edit` (if user message and capability)    |
| `r`             | `regenerate` (if assistant and capability) |
| `↑` / `↓`       | move focus (no intent emitted)             |
| `Enter`         | activate the focused control               |

Configurable in v2; fixed in v1.

## Testing

- **Pure unit tests** on `core/intents.ts` ensure synthesizers produce well-formed intents.
- **Adapter contract tests** assert each declared capability has a working `handleIntent` branch.
- **Mock adapter** in renderer tests records intents to a queue for assertion — no site needed.
