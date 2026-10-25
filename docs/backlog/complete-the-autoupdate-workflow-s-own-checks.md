---
kind: change
waits_on: []
design_with: []
recorded_by: open-autoupdate-pr-with-app-token
re_checked: 2026-10-19
---

`open-autoupdate-pr-with-app-token` added a section to `.github/tests/test_ci_configuration.py` that finds pull-request-opening workflow steps and asserts the identity they use. Its helpers have four reading limits, each producing a false positive on a legitimate change: `token_inputs` reads only step-level `env:` and misses a job- or workflow-level `GH_TOKEN` (the likeliest to bite, the first time a workflow opens a pull request with `gh`); `step_opens_a_pull_request` matches a POST and `/pulls` anywhere in one `run:` block rather than one command; `secrets_referenced_by` scans the whole job rather than the identity path; `readme_sections` splits at any `#` line, including inside a code fence.

The section also asserts nothing about two properties of `.github/workflows/pre-commit-autoupdate.yml`: the minting step's `permission-contents: write` / `permission-pull-requests: write` down-scoping, and the restricted input set on the pull-request step (no `labels`/`assignees`, which need Issues). Neither is live while the App's own scope is exactly those two permissions. Both come from a prose clause, not a scenario, so a scenario must be written before a test can be derived. The workflow header states which of its claims the suite does not stand behind; that note is what this change replaces.

- **Kind**: It owes a delta adding scenarios for the two unasserted properties, and it edits tests an independent author derived from delta specs.
- **Recorded by**: `open-autoupdate-pr-with-app-token`, whose tests were independently derived from its deltas; correcting or extending them was a change of its own. Merged here on 2026-09-16 from `widen-what-the-pull-request-identity-checks-can-read` and `assert-the-autoupdate-workflow-s-two-unchecked-properties`.
