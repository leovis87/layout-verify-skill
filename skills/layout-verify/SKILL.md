---
name: layout-verify
description: Use when changing web frontend layout — spacing, alignment, sizing, tab/card/bubble structure — or when the user reports a visual defect ("these overlap", "this is off-center", "the gap is wrong", "the sizes don't match"). Verifies completion by reading the browser's computed box model with a script rather than eyeballing screenshots, and turns each reported defect into a reusable invariant. Also applies to diagnosing the cause of a layout complaint, not just fixing it.
---

# layout-verify — finish layout work with measured pixels, not screenshots

> If your project has its own skill by this name, that one wins (it will have
> the concrete server URL, credentials, and script paths). This file is the
> general-purpose original.

## When to use this

- Any web frontend change to spacing, alignment, size, or the structure of
  tabs / cards / list rows
- Any user report of a visual defect — **start the diagnosis with measurement**,
  not with a guess about the cause
- Verifying that two states that should match actually match (tab A vs. tab B,
  sibling cards, rows in a list)

## Why this exists

Layout work by an LLM agent tends to get marked "done" because a screenshot
*looks* fine. What follows is a familiar loop: overlapping cards, a 1px height
jitter when switching tabs, asymmetric margins that only appear when a
conditional element renders empty. These regress no matter how many prose
conventions get written down, and eventually the user opens devtools and points
at the offending margin themselves — which is a failure.

The fix is to stop relying on human vision at the end of the loop: have a
script read what the browser actually computed, and let the person read numbers.

## Core principles

1. **Don't judge from screenshots.** Read the computed box model
   (`margin`/`padding`/`width`/`height`/`top`/`bottom`) via `getComputedStyle`
   or `boundingBox`, and put those numbers in your completion report.
2. **"1px off" is not within tolerance.** For states that come in pairs (before
   and after a tab switch, sibling cards, rows of the same list) the bar is
   **diff 0**. Signing off on "close enough" is how you get re-reported with
   the user's own measurements.
3. **Spacing is owned by the parent.** Gaps between elements inside a card or
   bubble belong on the parent (`space-y-*` / `gap-*` or the CSS equivalent),
   not as `mt-*` / `mb-*` on children. Child margins combine with conditional
   rendering to produce asymmetry — the root of the recurring pattern. If a
   list is empty, don't render the element at all: **a zero-height element
   still consumes the parent's gap.**
4. **One report = one invariant.** When the user reports a layout defect, the
   fix commit also adds that case to an invariant script — expressed as a rule
   that must hold for *any* input, not as a check for that one case. The goal
   is never receiving the same report twice.

## Procedure

1. Make the change, then start a dev/verification server (not production).
2. If the project has no invariant script yet, create one from the template
   below; otherwise add the invariants relevant to this change, then run it:
   ```
   node scripts/layout_invariants.cjs
   ```
   If the app needs a login, inject credentials via environment variables —
   never hardcode them.
3. **All PASS → report completion with the measured numbers.** Any FAIL →
   report the cause *with numbers*, fix, re-run. Don't commit or open a PR
   while red.
4. If you changed a screen the script doesn't cover, measure just those
   elements ad hoc and include the numbers. If it was a user-reported defect,
   promote it to a permanent invariant.

## Invariant script template

```js
// scripts/layout_invariants.cjs — node scripts/layout_invariants.cjs
const { chromium } = require('playwright')
const BASE = process.env.LAYOUT_BASE_URL || 'http://localhost:5173'

/** Each invariant: run(page) → { ok, detail }. detail MUST carry the numbers. */
const INVARIANTS = [
  {
    name: 'example: card height identical across tab switch',
    async run(page) {
      const h = async () => (await page.locator('#target-card').boundingBox()).height
      await page.getByRole('tab', { name: 'A' }).click()
      const a = await h()
      await page.getByRole('tab', { name: 'B' }).click()
      const b = await h()
      return { ok: a === b, detail: `${a}px vs ${b}px (diff ${Math.abs(a - b)}px)` }
    },
  },
  // Add one entry per user-reported defect:
  // - sibling cards share a uniform margin-bottom
  // - first/last child vertical padding is symmetric inside a bubble
  // - a conditional element that renders empty leaves no residual margin
]

;(async () => {
  const browser = await chromium.launch()
  const page = await browser.newPage({ viewport: { width: 390, height: 900 } })
  await page.goto(BASE) // handle login here via env-var credentials if needed
  let failed = 0
  for (const inv of INVARIANTS) {
    try {
      const { ok, detail } = await inv.run(page)
      console.log(`${ok ? 'PASS' : 'FAIL'}  ${inv.name} — ${detail}`)
      if (!ok) failed++
    } catch (e) {
      console.log(`FAIL  ${inv.name} — threw: ${e.message}`)
      failed++
    }
  }
  await browser.close()
  process.exit(failed === 0 ? 0 : 1)
})()
```

Playwright is the reference here, but nothing about the approach is
Playwright-specific — any driver that can read computed styles works.

## Recurring patterns worth encoding as invariants

- **Margin lost when merging siblings.** Two cards each carried `mb-*`; you
  wrap them in a shared container and drop those margins, but don't move the
  margin onto the wrapper — the merged card now collides with the next one.
  Test: *does the sum of the outer margins the siblings used to own still live
  somewhere after the merge?*
- **Paired-state height jitter.** For tabs/toggles, the "empty or collapsed"
  state on both sides needs the same `min-h-*`, or the container resizes on
  every switch.
- **Conditional render + child margin.** When the element above renders empty,
  only its margin survives — asymmetry.
- **Per-column `nowrap` in tables.** `white-space: nowrap` on the first column
  means one long sentence lets that column eat the full width and collapses the
  rest. Keep `nowrap` on headers only; let body cells wrap naturally (for CJK,
  add `word-break: keep-all`).
- **Tooltips/popovers inside a scroll container.** An `absolute` tooltip is
  clipped by any ancestor with `overflow: auto|hidden|scroll` — and setting
  only `overflow-y` still clips horizontally, so moving it sideways doesn't
  help. `z-index` is irrelevant (it's clipping, not stacking). Use `fixed` with
  coordinates computed on hover, and measure the tooltip's box against the
  viewport on **both the first and last row** — they overflow in opposite
  directions.

## What this skill does not do

- Doesn't enforce itself via a commit hook — it needs a running server, which
  makes hooks flaky. This is a working procedure, not a commit gate.
- Doesn't run against production.
- Doesn't state a cause without measuring it. No numbers means it's a guess,
  not a diagnosis.
- Doesn't substitute for real-device sign-off where the emulator can't
  reproduce the behavior (mobile keyboards, dynamic viewports).

## Status

This was extracted from repeated real layout regressions in one project —
enough of them that prose conventions demonstrably weren't holding, which is
what motivated mechanizing the check. The procedure and the pattern catalog
reflect that history plus reasoning about what generalizes; the template script
is deliberately minimal and expects you to grow it.

It has **not been exercised across many independent codebases**, and the
"parent owns spacing" principle is stated in Tailwind-flavored terms because
that's where the cases came from — the underlying rule is framework-agnostic
but the class names in the examples are not.

If you use it, please open an issue or PR with what worked, what it missed, and
which patterns you had to add.
