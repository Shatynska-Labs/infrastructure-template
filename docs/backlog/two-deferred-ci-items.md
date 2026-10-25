---
kind: change
waits_on: []
design_with: []
recorded_by: close-ci-verification-gaps
re_checked: 2026-10-19
---

`close-ci-verification-gaps` deferred two unrelated CI improvements:

- **The destroy-policy gate's inspection logic is inline workflow shell.** Moving it into a version-controlled script with executable fixtures would make the highest-consequence logic in this repository testable as code. That change's `design.md` Decision 5 deferred it on scope, not on merit. Its `tasks.md` 1.1 describes four fixtures: clean, destructive, malformed, and valid JSON that is not a plan. `.github/tests/test_ci_configuration.py` asserts that the routes are closed, not that each one is reached.
- **`actionlint` is cited as a verification means, but nothing installs it.** It is not in `.pre-commit-config.yaml`. Adding the hook first needs a decision on two existing, intentional `SC2016:info` findings: single-quoted markdown in an `echo`, in `pr-validation.yml`'s plan-comment step and in `apply.yml`'s job-summary step.

A third item was taken as a fix on 2026-09-15: `pre-commit-autoupdate.yml` now installs the pinned `pre-commit` from `.github/requirements-ci.txt`.

- **Kind**: The first item restructures the production destroy gate, and the second needs an operator decision on how to handle the two findings.
- **Recorded by**: `close-ci-verification-gaps`, which deferred both on scope rather than rejecting them.
