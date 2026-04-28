# Risk Register

Risks rated by **likelihood × impact** on a 1–3 scale. Mitigations are concrete actions, not aspirations.

## R1 — Site updates break adapter parsing

- **Likelihood**: 3 (frequent — both sites ship multiple times per week)
- **Impact**: 2 (visible breakage on supported site, but other site and renderer unaffected)
- **Mitigation**:
  - Strategy A (network interception) is more stable than DOM scraping; prefer it.
  - Snapshot tests on captured payloads detect regressions in CI before users see them — but only catch shapes we've captured, not new ones.
  - Capture a fresh fixture from each site weekly (manual, ~5 min) and run snapshot suite.
  - Adapters fail soft: on parse error, the affected message renders as a single opaque HTML block from the site's original DOM. User sees a degraded but readable view; we get a parse-error report.
  - Bug-report intent (`reportBug` with `kind: 'parse'`) gives us a low-friction reporting channel.

## R2 — Pretext doesn't match site typography exactly

- **Likelihood**: 3 (subtle differences will exist)
- **Impact**: 2 (users notice; not a functional break)
- **Mitigation**:
  - Match host font, line-height, letter-spacing per site by reading computed styles from the original message list at startup.
  - Side-by-side visual diff page (`test/visual-diff.html`) shows our render next to the original on a fixture set.
  - Iterate until "indistinguishable at a glance" — perfection is not a goal; close-enough is.
  - User-visible "show original" toggle as escape hatch if our rendering is unacceptable for a specific case.

## R3 — Performance gains don't materialize on real corpora

- **Likelihood**: 1 (M0 prototype validates this before commitment)
- **Impact**: 3 (kills the project)
- **Mitigation**:
  - M0 milestone is explicitly a go/no-go gate. If 5000 messages don't scroll smoothly, we re-evaluate before adapter work.
  - Profile with real corpora, not just synthetic fixtures. Solicit conversation exports from dogfooders early.
  - If pretext alone isn't enough, fall back to `content-visibility: auto` augmentation — a less ambitious win that still helps users.

## R4 — KaTeX / image / highlight async work blocks first paint

- **Likelihood**: 2
- **Impact**: 2 (visible "loading" stutter on heavy conversations)
- **Mitigation**:
  - Initial paint uses placeholder dimensions for opaque blocks; real measurements arrive via `replace` patch and trigger re-layout below the fold.
  - Highlight in-viewport blocks synchronously; defer rest to `requestIdleCallback`.
  - KaTeX cache is global; identical formulas are free after first render.
  - Acceptance: first paint of viewport content within 200 ms of conversation load.

## R5 — Streaming patch storm starves the renderer

- **Likelihood**: 1
- **Impact**: 2 (frame drops during fast streaming)
- **Mitigation**:
  - Per-frame patch batching with coalescing (described in `05-streaming.md`).
  - Backpressure: defer non-visible message updates when queue depth is high.
  - Frame-budget regression tests in CI.

## R6 — Selection / copy doesn't match user expectation

- **Likelihood**: 2
- **Impact**: 2 (frustrating; common workflow)
- **Mitigation**:
  - Native browser selection over pretext-rendered spans works by default.
  - Copy from inside a code block returns `CodeBlock.source` exactly (overrides default selection text).
  - Multi-block / multi-message copy uses DOM concat + `OpaqueBlock.copyText` overrides.
  - Test plan covers: copy a single line of code, copy a whole code block, copy across messages, copy math.

## R7 — Cmd+F (browser find) only sees mounted content

- **Likelihood**: 3 (this is a known limitation of any virtualized list)
- **Impact**: 1 (annoying but workable; users have other paths)
- **Mitigation**:
  - v1: ship a custom search UI in renderer chrome; document the limitation.
  - v2: temporarily mount full conversation into hidden DOM on Cmd+F; revert on Esc. Deferred because the heuristics are fiddly.
  - Mark as a known issue in user docs; not a blocker.

## R8 — Accessibility regression vs. native site

- **Likelihood**: 2
- **Impact**: 2 (excludes screen reader users from improvements)
- **Mitigation**:
  - Renderer mounts `role="article"` per message with author label.
  - "Render full transcript" toggle bypasses virtualization for screen reader / print sessions.
  - The original site's message list stays in DOM (hidden via CSS); a future a11y mode could un-hide it for assistive tech only.
  - A11y pass before public release; not optional.

## R9 — Page CSP blocks our content script

- **Likelihood**: 1 (MV3 content scripts are exempt from page CSP for `js` files)
- **Impact**: 3 (extension can't run)
- **Mitigation**:
  - No inline scripts, no eval — already a CSP-clean codebase.
  - Trusted Types policy registered for HTML insertion.
  - Test on each site's strictest CSP environment before each release.

## R10 — User loses trust in extension after bad parse

- **Likelihood**: 2
- **Impact**: 3 (uninstall — hard to recover)
- **Mitigation**:
  - Threshold gating (don't activate on conversations under 50 messages) limits exposure on sensitive new chats.
  - Fail-soft on parse error: render affected message from original DOM as opaque block.
  - Easy in-page toggle to disable for current site without going to options.
  - Conservative dogfooding before public launch.

## R11 — Pretext upstream changes break our integration

- **Likelihood**: 1 (small library, slow-moving)
- **Impact**: 2 (touches `core/layout.ts` only)
- **Mitigation**:
  - Pin pretext to a specific version; vendor a copy in `vendor/pretext/` if needed.
  - Read-through API tests against the version we pin; bump deliberately.

## R12 — Legal / ToS concerns from sites

- **Likelihood**: 2
- **Impact**: 3 (cease and desist; store removal)
- **Mitigation**:
  - We don't republish, redistribute, scrape at scale, or send conversation data anywhere.
  - All processing is client-side, in-page.
  - We don't bypass auth, paywall, or rate limits.
  - Open source the extension; users install knowingly.
  - Do not use site logos or claim affiliation.
  - Have a plan to migrate to a userscript distribution if extension stores remove us.

## What is NOT on this list

- "What if pretext is buggy?" — same as any dependency; tested in M0, pinned, reviewed.
- "What if Chrome deprecates MV3?" — Manifest V4 will exist; rewrites are a year away if it happens.
- "What if a user has 100k messages?" — virtualization scales; this is the easiest case, not the hardest.
