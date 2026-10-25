---
kind: change
waits_on: []
design_with: []
recorded_by: apply-host-configuration-through-a-gated-workflow
re_checked: 2026-10-19
---

`pr-validation.yml`, `apply.yml` and `drift.yml` share one stack-discovery shell body of about 160 lines. `.github/tests` asserts that the three copies are identical: it locates each body by the `terraform/stacks` path literal it contains and executes it against a scratch tree. `host-converge.yml` and `platform-deploy.yml` now carry similar bodies that are deliberately not identical, and both sit outside the identity assertion. `host-converge.yml` enumerates `ansible/inventory/*.hcloud.yml` and cross-checks it against the stacks, and `platform-deploy.yml`'s comment counts "Five near-siblings".

A script under `.github/scripts/` (none exists yet) is likely simpler than a composite action, because the tests would execute it directly instead of locating it by a path literal. The shared parts are the flat `key: value` reader, the secret-name validation, the empty-result refusal and the JSON emission. The roots and the cross-checks stay parameters. The cost to weigh is that the diff restructures the production apply path and must replace the identity assertion rather than retarget it.

- **Kind**: It restructures every gated workflow's discovery and replaces a test assertion, which is too large for one review sitting.
- **Recorded by**: `apply-host-configuration-through-a-gated-workflow`, which wrote the fourth body (`host-converge.yml`). Factoring was out of scope for a change that was already restructuring gated workflows.
