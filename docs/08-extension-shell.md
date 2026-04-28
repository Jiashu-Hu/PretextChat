# Extension Shell (`src/shell/`)

The browser extension wrapper that wires renderer + adapter into the page. Minimal surface; nearly all the interesting code lives elsewhere.

## Manifest (MV3)

```json
{
  "manifest_version": 3,
  "name": "Chat Pretext",
  "version": "0.1.0",
  "description": "Fast rendering for long ChatGPT and Grok conversations.",
  "permissions": ["storage"],
  "host_permissions": [
    "https://chatgpt.com/*",
    "https://chat.openai.com/*",
    "https://grok.com/*",
    "https://x.com/i/grok*"
  ],
  "background": { "service_worker": "background.js" },
  "content_scripts": [
    {
      "matches": [
        "https://chatgpt.com/*",
        "https://chat.openai.com/*",
        "https://grok.com/*",
        "https://x.com/i/grok*"
      ],
      "js": ["content.js"],
      "run_at": "document_start",
      "all_frames": false,
      "world": "ISOLATED"
    },
    {
      "matches": [
        "https://chatgpt.com/*",
        "https://chat.openai.com/*",
        "https://grok.com/*",
        "https://x.com/i/grok*"
      ],
      "js": ["page-hook.js"],
      "run_at": "document_start",
      "all_frames": false,
      "world": "MAIN"
    }
  ],
  "action": { "default_popup": "popup.html" },
  "options_page": "options.html",
  "icons": { "16": "icon16.png", "48": "icon48.png", "128": "icon128.png" }
}
```

Notes:
- **Two content scripts**, two worlds. `page-hook.js` runs in `MAIN` world so it can monkey-patch `window.fetch` before the page's own scripts run. It posts intercepted payloads to the ISOLATED-world `content.js` via `window.postMessage`.
- **Minimum permissions**: only `storage` (for per-site enable + theme override). No `tabs`, no `webRequest`, no `<all_urls>`.
- **No remote code**: all bundles ship with the extension; no `eval`, no remote scripts.

## Content-script entry (`shell/content.ts`)

```ts
import { createRenderer } from '../core'
import { adapters } from '../adapters'

const THRESHOLD = 50  // see "Hidden by default for new conversations" below

// Active session state (renderer + DOM mount + cache observer) for the current
// conversation, if any. Null on below-threshold or unsupported pages.
type Session = { id: number; teardown: () => void }
let active: Session | null = null
let nextSessionId = 0

async function main() {
  // Tear down any prior session first; both above- and below-threshold paths
  // install something that needs cleaning before a new conversation starts.
  active?.teardown()
  active = null

  // Per-invocation token. Every await below can race with another navigation
  // re-entering main(); on resume we check `mySession === nextSessionId - 1`
  // (i.e., we are still the most recent caller) before applying any side
  // effect or before treating a thrown error as relevant. Without this, a
  // late rejection from an obsolete start() can teardown the *current*
  // session and the user's new conversation goes blank.
  const mySession = ++nextSessionId
  const isCurrent = () => mySession === nextSessionId

  const adapter = adapters.find(a => a.matches(location, document))
  if (!adapter) return  // not a supported site

  const convoId = adapter.parseConversationIdFromUrl(new URL(location.href))
  const cachedCount = convoId ? await getCachedCount(adapter.name, convoId) : 0
  if (!isCurrent()) return                          // navigated away mid-await
  const shouldMount = cachedCount >= THRESHOLD

  if (!shouldMount) {
    const stop = convoId ? observeAndCache(adapter, convoId) : () => {}
    active = {
      id: mySession,
      teardown: () => { stop(); adapter.dispose() },
    }
    return
  }

  const container = await waitForChatContainer(adapter)
  if (!isCurrent()) { adapter.dispose(); return }

  // Resolve theme BEFORE we touch the native UI. A pending await between
  // hide-time and `active` registration would leave `cpHidden` stuck if the
  // session were superseded or the await rejected.
  const theme = await getTheme()
  if (!isCurrent()) { adapter.dispose(); return }

  const original = adapter.findMessageList(container)
  original.dataset.cpHidden = '1'

  const mount = document.createElement('div')
  mount.id = 'cp-mount'
  original.parentElement!.insertBefore(mount, original)

  const renderer = createRenderer({
    container: mount,
    adapter,
    theme,
  })

  // Register teardown BEFORE awaiting start(): if start() throws or this
  // session is superseded, we still need to un-hide native, remove mount,
  // dispose adapter, stop renderer. Invariant: while anything is hidden or
  // mounted, `active` holds a teardown that restores the page.
  const session: Session = {
    id: mySession,
    teardown: () => {
      renderer.stop()
      delete original.dataset.cpHidden
      mount.remove()
      adapter.dispose()
    },
  }
  active = session

  try {
    await renderer.start()
  } catch (err) {
    // Only act if we are still the current session. Otherwise the rejection
    // belongs to a stale invocation and a newer main() already owns `active`.
    if (active?.id === session.id) {
      console.error('[chat-pretext] renderer.start() failed; restoring native UI', err)
      session.teardown()
      active = null
    }
    return
  }

  if (!isCurrent()) {
    // A newer main() ran while start() was pending. Tear ourselves down
    // (we still own `session.teardown`) without touching `active`.
    session.teardown()
    return
  }

  if (convoId) renderer.onMessageCountChange(n => setCachedCount(adapter.name, convoId, n))
}

// SPA navigation handler is installed ONCE at script load, outside main(),
// so it fires regardless of whether the previous main() above- or
// below-threshold-returned. Without this, landing on a short conversation
// and SPA-navigating to a long one would leave the user on the slow native UI.
observeRouteChanges(() => { main().catch(err => console.error('[chat-pretext]', err)) })

main().catch(err => console.error('[chat-pretext]', err))
```

The cache is a small `chrome.storage.local` map keyed by `${adapterName}:${convoId}` → message count. Total size is bounded by the user's conversation count; even a power user with thousands of threads stays under a few hundred KB.

**Why decide before hiding rather than mount-then-fallback:** flipping back from "mounted empty" to "native UI" creates a visible flash on every short conversation. Deferring on first visit costs the user nothing on a thread that wouldn't have wanted the renderer anyway, and only delays renderer activation by one visit on threads that grow into needing it — by which point the next visit is instant.

## Page-hook (`shell/page-hook.ts`) and IPC

Runs in `MAIN` world. Monkey-patches network APIs and forwards intercepted streams to the ISOLATED-world adapter over a private `MessageChannel`.

### Why a MessageChannel, not raw `window.postMessage`

`window.postMessage` is observable by every script in the page — including other extensions injecting MAIN-world scripts and any third-party scripts the host page loads. Using it for stream chunks would let any such script forge adapter input. A `MessageChannel` confines per-message traffic to the two endpoints holding port references; raw `postMessage` does not.

### Honest threat model (the limitations `MessageChannel` does *not* fix)

MV3 does not provide a fully private MAIN ↔ ISOLATED IPC primitive. There is no DOM mechanism that transfers a live JS object reference (such as a `MessagePort`) between worlds — `postMessage` is the only path, and `postMessage` events are visible to every same-window listener. Two distinct attacks follow:

1. **Bootstrap eavesdropping by other extensions.** When MAIN posts the bootstrap with `port2` in the transfer list, the event dispatches to all `message` listeners on `window`, including ISOLATED-world content scripts of other installed browser extensions. Any such listener can call `event.ports[0]` and retain a reference to the port. We cannot prevent this; it is intrinsic to how `postMessage` works across the MAIN/ISOLATED boundary.

2. **Bootstrap forgery by page scripts.** `bootstrapSecret` is written to a DOM dataset attribute so MAIN can read it. Any same-origin page script (or third-party script loaded by the host page) can read that attribute too, then race ahead of `page-hook.js` and post its own `cp:bootstrap` message with its own `MessagePort`. ISOLATED's listener has no way to distinguish the forged port from the real one — it accepts the first valid-looking bootstrap and removes itself, after which the real `page-hook.js` cannot bind. The "secret" therefore prevents *accidental* message-tag collisions but does **not** authenticate the sender.

What this means in practice:
- **Defended:** the host page reading conversation bytes (it already could — we only forward what the page itself fetched), later-loading third-party scripts that miss the bootstrap event entirely.
- **Not defended against:** other browser extensions in the MAIN/ISOLATED worlds, and same-origin page scripts that can race ahead of `page-hook.js`.

Defense in depth — and the **only** real defense — is at the adapter:

- The adapter treats every byte arriving on the port as untrusted input. It validates the parsed payload shape (status codes, JSON envelope, message shape, schema-version compatibility) before producing patches. A forged stream produces a parse failure, not silent corruption. This requirement is canonical in [`CONTRACTS.md`](CONTRACTS.md) under "Adapter input is untrusted."
- We do not transmit anything sensitive over the port — only response bytes the host site already exposed to the page itself. A successful eavesdropper learns nothing it could not also learn by attaching its own `fetch` interceptor to the page.
- A successful forger can degrade UI quality (e.g., make us render garbage that we then reject) but cannot cause data loss elsewhere — the native UI still has the truth, and the renderer's failure mode is to show its own error affordance, not corrupt the page.

This trade-off is consistent with how production extensions (password managers, ad blockers, etc.) handle MV3 cross-world IPC. Treating MAIN-world data as untrusted is the standard discipline.

### Why a DOM-attribute rendezvous

MV3 does not guarantee the relative load order of MAIN-world and ISOLATED-world `document_start` content scripts. A one-shot `postMessage` from whichever side runs first will be lost if the other side hasn't installed its listener yet. To make the handshake order-independent, ISOLATED **advertises readiness** via a DOM dataset attribute *after* installing its bootstrap listener; MAIN **observes** that attribute (synchronously on load, then via `MutationObserver` if not yet present) and only sends the bootstrap once it sees ISOLATED is ready. Because ISOLATED writes the attribute strictly after installing its listener, MAIN's bootstrap message is guaranteed to land.

### Handshake (ISOLATED side, in `shell/content.ts`)

```ts
// 1. Install the bootstrap listener BEFORE advertising readiness, so
//    no matter when MAIN posts the bootstrap, we are already listening.
let hookPort: MessagePort | null = null
const bootstrapSecret = crypto.randomUUID()

window.addEventListener('message', function bootstrap(e) {
  if (e.source !== window) return
  if (e.origin !== location.origin) return
  if (e.data?.tag !== 'cp:bootstrap') return
  if (e.data.secret !== bootstrapSecret) return       // forged bootstrap
  if (e.ports.length !== 1) return
  hookPort = e.ports[0]
  hookPort.start()
  hookPort.onmessage = (ev) => routeFromHook(ev.data)
  window.removeEventListener('message', bootstrap)    // one-shot
}, { capture: true })

// 2. Now signal our presence to MAIN. The attribute carries the secret
//    so MAIN can authenticate its bootstrap without a separate channel.
//    Writing happens AFTER step 1, so any MAIN that observes the attribute
//    is guaranteed to find a live listener on this side.
document.documentElement.dataset.cpIsolated = bootstrapSecret
```

### Page-hook (MAIN side, `shell/page-hook.ts`)

```ts
let toIsolated: MessagePort | null = null
// Resolves when the port is bound. Early streams await this before
// switching to live forwarding (see pumpEarly below).
let portReady: Promise<MessagePort>
let portReadyResolve!: (p: MessagePort) => void
portReady = new Promise<MessagePort>(r => (portReadyResolve = r))

function bindIsolated(port: MessagePort) {
  toIsolated = port
  toIsolated.start()
  portReadyResolve(port)
}

function tryBootstrap(): boolean {
  const secret = document.documentElement.dataset.cpIsolated
  if (!secret) return false                            // ISOLATED not ready
  const channel = new MessageChannel()
  // ISOLATED's bootstrap handler claims port2 synchronously; no ack needed.
  window.postMessage(
    { tag: 'cp:bootstrap', secret },
    location.origin,
    [channel.port2],
  )
  bindIsolated(channel.port1)
  return true
}

if (!tryBootstrap()) {
  // ISOLATED ran second (or hasn't run yet). Watch the documentElement
  // for the readiness attribute.
  const obs = new MutationObserver(() => {
    if (tryBootstrap()) obs.disconnect()
  })
  obs.observe(document.documentElement, {
    attributes: true,
    attributeFilter: ['data-cp-isolated'],
  })
}

const origFetch = window.fetch
window.fetch = async function (...args) {
  const res = await origFetch.apply(this, args)
  // fetch() accepts string | URL | Request — normalize all three.
  const input = args[0]
  const url =
    typeof input === 'string' ? input
    : input instanceof URL    ? input.href
    : input instanceof Request ? input.url
    : ''
  if (looksInteresting(url) && res.body) {
    const [forUs, forSite] = res.body.tee()
    if (toIsolated) {
      pumpToIsolated(url, forUs)
    } else {
      // Handshake hasn't completed. Start an active reader IMMEDIATELY:
      // ReadableStream.tee() buffers an unread branch internally as the
      // other branch is consumed by the page, so leaving forUs idle
      // would accumulate bytes. pumpEarly drains forUs into a bounded
      // queue and switches to live forwarding once the port is bound.
      pumpEarly(url, forUs)
    }
    return new Response(forSite, res)
  }
  return res
}

const MAX_EARLY_BUFFER_BYTES = 5 * 1024 * 1024   // 5 MB cap per pre-handshake stream
const HANDSHAKE_TIMEOUT_MS = 5000                // give up if no port by this point

async function pumpEarly(url: string, body: ReadableStream<Uint8Array>) {
  const reader = body.getReader()
  const buffered: Uint8Array[] = []
  let bufferedBytes = 0
  const streamId = crypto.randomUUID()

  // If the port arrives during buffering, switch to live mode immediately.
  const handoff = portReady.then(port => ({ port }))
  // Or give up after the handshake timeout.
  const timeout = new Promise<{ timeout: true }>(r =>
    setTimeout(() => r({ timeout: true }), HANDSHAKE_TIMEOUT_MS),
  )

  try {
    while (true) {
      const next = reader.read()
      const winner = await Promise.race([
        next.then(v => ({ chunk: v })),
        handoff,
        timeout,
      ])

      if ('timeout' in winner) {
        // Handshake didn't arrive in time. Cancel forUs so tee stops
        // accumulating; the adapter will need to re-fetch history if it
        // cares. (For ChatGPT/Grok the SSE turn that triggered this is
        // also re-derivable from the next stream, so the loss is bounded.)
        await reader.cancel('cp:handshake-timeout')
        console.warn('[chat-pretext] handshake timeout; dropped early stream', url)
        return
      }

      if ('port' in winner) {
        // Port is live. Replay buffered chunks then continue in live mode.
        const port = winner.port
        port.postMessage({ tag: 'cp:stream-open', streamId, url })
        for (const c of buffered) port.postMessage({ tag: 'cp:stream-chunk', streamId, chunk: c })
        // Drain the rest of the stream directly to the port.
        // (We may still have an in-flight `next` read; await and forward it.)
        const tail = await next
        if (!tail.done && tail.value) {
          port.postMessage({ tag: 'cp:stream-chunk', streamId, chunk: tail.value })
        }
        while (true) {
          const r = await reader.read()
          if (r.done) break
          port.postMessage({ tag: 'cp:stream-chunk', streamId, chunk: r.value })
        }
        port.postMessage({ tag: 'cp:stream-end', streamId })
        return
      }

      // Pre-handshake chunk. Append to bounded buffer.
      const { chunk } = winner
      if (chunk.done) {
        // Stream ended before port arrived. Wait for port (or timeout)
        // and replay only the buffered chunks.
        const p = await Promise.race([handoff, timeout])
        if ('timeout' in p) {
          console.warn('[chat-pretext] handshake timeout after stream end; dropped buffered stream', url)
          return
        }
        const port = p.port
        port.postMessage({ tag: 'cp:stream-open', streamId, url })
        for (const c of buffered) port.postMessage({ tag: 'cp:stream-chunk', streamId, chunk: c })
        port.postMessage({ tag: 'cp:stream-end', streamId })
        return
      }
      bufferedBytes += chunk.value!.byteLength
      if (bufferedBytes > MAX_EARLY_BUFFER_BYTES) {
        await reader.cancel('cp:buffer-limit')
        console.warn('[chat-pretext] early stream exceeded buffer cap; dropped', url)
        return
      }
      buffered.push(chunk.value!)
    }
  } catch (err) {
    console.error('[chat-pretext] pumpEarly error', err)
    try { await reader.cancel(String(err)) } catch {}
  }
}

async function pumpToIsolated(url: string, body: ReadableStream<Uint8Array>) {
  const port = toIsolated!
  const streamId = crypto.randomUUID()
  port.postMessage({ tag: 'cp:stream-open', streamId, url })
  const reader = body.getReader()
  try {
    while (true) {
      const { value, done } = await reader.read()
      if (done) break
      // Uint8Array clones cleanly across the port. We never post the
      // ReadableStream object itself: structured-clone support for
      // ReadableStream over postMessage is unreliable in MV3 today.
      port.postMessage({ tag: 'cp:stream-chunk', streamId, chunk: value })
    }
    port.postMessage({ tag: 'cp:stream-end', streamId })
  } catch (err) {
    port.postMessage({ tag: 'cp:stream-error', streamId, message: String(err) })
  }
}

function looksInteresting(url: string) {
  return /\/backend-api\/conversation/.test(url)   // ChatGPT
      || /\/grok\/api\/conversation/.test(url)     // Grok (placeholder)
}
```

The `tee()` is critical — we must not drain the response body before the page reads it.

### Routing in ISOLATED

`hookPort.onmessage` accumulates chunks per `streamId` and exposes each open stream to the active adapter as an `AsyncIterable<Uint8Array>` (one value per `cp:stream-chunk`, terminating on `cp:stream-end`, throwing on `cp:stream-error`). Because the channel is private, no per-message origin/source validation is needed — only the two endpoints hold a port reference.

**Known limitation: synthetic `Response` metadata.** `new Response(forSite, res)` propagates `status`, `statusText`, and `headers` (those are in `ResponseInit`), but `url`, `redirected`, and `type` are not exposable via any public API on `Response`. After `tee()` the original body is locked, so we cannot return `res` itself. For ChatGPT and Grok this is harmless — neither reads those fields on the conversation endpoint, and it is part of the v1 acceptance check that we audit each new site for this. If a future adapter targets a site that *does* read them, the fallback is to skip interception on that endpoint and re-fetch from the content script with the same auth headers (one extra round-trip, metadata-clean).

## Background service worker (`shell/background.ts`)

Minimal. Only needed for:
- Storage helpers (per-site enable, theme override).
- Opening the options page.
- Lifecycle: install / update notifications.

No conversation data passes through the service worker.

## Options page (`shell/options.html`)

Simple HTML form bound to `chrome.storage.local`:

- Toggle per site: ChatGPT, Grok.
- Theme: auto / light / dark.
- Debug mode (shows perf overlay + verbose logging).
- "Render full transcript" default (accessibility).

## SPA navigation handling

Both supported sites are React SPAs that swap conversations without full page reloads. The shell:

- Watches `history.pushState` / `replaceState` (patched in page-hook for synchronous detection) and `popstate`.
- On detected navigation, calls `renderer.stop()` and re-enters `main()` after a short debounce.
- Adapters expose `parseConversationIdFromUrl(url)` so the renderer can show a stable placeholder during the swap (avoids flash of empty state).

## Selection between us and the site

The original site's message list stays in the DOM (hidden via CSS). This costs us memory but buys:
- Native browser features (find-in-page, screen reader fallback) still work on the original.
- A debug "show original" toggle is one CSS rule away.
- Fallback if the renderer crashes mid-session — un-hide and the user keeps working.

In v2 we may detach the original entirely once stable; for v1, keep it.

## Hidden by default for new conversations

A fresh conversation with one or two messages doesn't need virtualization. The renderer activates only when the conversation crosses a message-count threshold (default 50, configurable in options). Below the threshold the original UI is shown unmodified — no risk on short threads where bugs would be most visible per-user.

The decision happens **before** the shell touches the native message list (see content-script entry above). The shell consults a per-conversation message-count cache keyed by `(adapter.name, parseConversationIdFromUrl(url))`. On the first visit to a new conversation the cache is empty, so the shell defers and observes the patch stream into the cache without hiding anything. Because conversations grow monotonically, every later visit gets an instant correct decision and the native list is hidden only when we know the renderer will mount.

## CSP and Trusted Types

Both target sites set strict CSPs. Our content script:
- Does not inject inline scripts; all code is bundled `js` files.
- Does not eval; no `Function()` constructor.
- Sanitizes adapter-emitted HTML through `services/sanitize.ts` before insertion.
- Uses `TrustedHTML` policies where the page enforces them.

Page-hook injection works because `MAIN`-world content scripts in MV3 are exempted from page CSP.
