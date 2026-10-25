---
kind: fix
waits_on: []
design_with: []
re_checked: 2026-10-19
---

`AGENTS.md` says "Do not hard-wrap prose. Keep each paragraph on a single line whatever its length", and nothing checks it. Hard wraps re-accumulate silently: the sweep beside which this was recorded rewrapped 72 of 211 tracked markdown files. Under the rule, any prose paragraph spanning more than one line is a violation, with no threshold to tune. The detector must exempt the breaks Markdown requires: fenced and indented code, tables, YAML frontmatter, list-item continuations, and a line ending in two spaces or a backslash.

Scope is markdown prose only; comments in `.py`, `.yml`, `.sh` and `.tf` files stay wrapped. It is not a formatter: Prettier's `proseWrap: "never"` would rewrite style across the tree and archived records, and there is no Node toolchain for it; markdownlint has no rule for the opposite of `MD013`. It fits `.github/tests` as one new module.

- **Kind**: A static assertion with no delta or decision.
- **Recorded by**: unknown change; recorded 2026-09-11 beside the fix that removed the hard wraps, which wrote the rule but not its check.
