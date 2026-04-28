# Overview

## Problem

ChatGPT and Grok become slow and janky in long conversations. The bottleneck is not text measurement — it is the sheer DOM node count, repeated syntax-highlighter passes, React reconciliation across thousands of messages, and the absence of viewport windowing. A 5000-message thread can stutter on scroll and lock the main thread on resize.

## Solution

A browser extension that replaces the chat message list with a custom renderer built on [`chenglou/pretext`](https://github.com/chenglou/pretext). Pretext lays out plain text without DOM reflow, so paragraphs and code blocks become cheap to measure and re-flow. Combined with viewport virtualization, the renderer keeps frame times stable regardless of conversation length.

The extension does **not** replace login, the model picker, the sidebar, or the composer. It only takes over the message list region.

## Goals (v1)

- Smooth scroll on conversations of 5000+ messages.
- Fast resize (column width changes in O(visible lines), not O(total lines)).
- Visual fidelity close enough that users do not notice the swap during normal reading.
- Two sites supported: ChatGPT and Grok.
- Read-mostly: full rendering and scrolling work; streaming for the active message works; site-native edit/regenerate buttons proxy to the original UI.

## Non-goals (v1)

- Replacing the composer or input box.
- Replacing the conversation sidebar.
- Replacing login, account, billing, or settings UI.
- Cross-site sync, history export, search across conversations — out of scope.
- Custom syntax themes beyond light/dark parity with the host site.
- Mobile browsers.

## Hard constraints

- **No 2D math layout.** LaTeX/KaTeX stays opaque (rendered HTML mounted as a block) — pretext is a 1D line layout engine and cannot render fractions, integrals, matrices.
- **No silent data egress.** The extension reads conversation data from the page or its network calls. It must not send conversation content anywhere off the user's machine.
- **No write actions on user behalf.** Edit, regenerate, feedback, and copy actions surface as user-triggered intents handed back to the site's own controls.

## Success criteria

- 60 fps scrolling on a 5000-message synthetic conversation on a mid-tier laptop.
- p95 resize re-layout under 16 ms when only viewport-visible blocks need re-measure.
- **Mounted DOM nodes** sub-linear in total messages — only viewport ± overscan are in the DOM at any time. Note that **in-memory message+block data is linear** in total content; the renderer holds the full `Message[]` for scroll-anchoring and offset bookkeeping. This is intentional: keeping parsed blocks in memory is cheap (a few hundred bytes per message) and is what makes virtualization fast. A 5000-message conversation runs at well under 50 MB of JS heap on the corpora we test.
- Adapter break from a site update is fixable in under a day, without renderer changes.
