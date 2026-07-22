# layout-verify

A [Claude Code](https://claude.com/claude-code) skill for finishing web layout
work against **measured pixels** instead of screenshots — and for turning each
reported visual defect into a reusable invariant.

## Why this exists

Layout work done by an LLM agent tends to get marked "done" because a
screenshot *looks* right. What follows is a familiar loop: overlapping cards, a
1px height jitter when switching tabs, asymmetric margins that only show up
when a conditional element renders empty. These keep regressing no matter how
many prose conventions you write down — and eventually the user opens devtools
and points at the offending margin themselves, which is the real failure.

So stop putting human vision at the end of the loop. Have a script read what
the browser actually computed, and let the person read numbers.

The skill sets four rules:

- **Don't judge from screenshots** — report the computed box model.
- **"1px off" is not tolerance.** For paired states (tab A vs. tab B, sibling
  cards, rows of a list) the bar is diff 0.
- **Spacing is owned by the parent**, not by child margins — and a zero-height
  element still eats the parent's gap, so don't render empty containers.
- **One defect report = one invariant**, written as a rule that must hold for
  any input rather than a check for that one case.

It ships a minimal invariant-script template (Playwright in the example, but
nothing about the approach is Playwright-specific) plus a catalog of the
patterns that actually recur — merged siblings losing a margin, paired-state
height jitter, per-column `nowrap` collapsing a table, tooltips clipped by a
scroll container.

## Install

Copy `skills/layout-verify/` into your Claude Code skills directory
(project-level `.claude/skills/` or user-level `~/.claude/skills/`), or use
this repo as a Claude Code plugin marketplace source.

If your project has its own skill by the same name, that one should win — it
will carry the concrete server URL, credentials, and script paths. This repo is
the general-purpose original.

## Status — help wanted

Extracted from repeated real layout regressions in one project — enough of them
that prose conventions demonstrably weren't holding, which is what motivated
mechanizing the check in the first place. The procedure and pattern catalog
reflect that history plus reasoning about what generalizes.

It has **not been exercised across many independent codebases.** The
"parent owns spacing" principle is stated in Tailwind-flavored terms because
that's where the cases came from — the underlying rule is framework-agnostic,
the class names in the examples are not. The template script is deliberately
minimal and expects you to grow it.

If you try it, please open an issue or PR with:
- what worked
- what the flow missed or got wrong
- which recurring patterns you had to add

## License

MIT
