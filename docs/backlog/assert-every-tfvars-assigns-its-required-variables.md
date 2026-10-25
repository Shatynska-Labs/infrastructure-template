---
kind: fix
waits_on: []
design_with: []
recorded_by: add-a-staging-environment
re_checked: 2026-10-19
---

Nothing checks that each stack's `terraform.tfvars` assigns every variable its `variables.tf` declares without a `default`. A gap surfaces only at plan time: `pr-validation.yml`, `apply.yml` and `drift.yml` run Terraform with `-input=false`, so the plan fails `No value for required variable`, the required check fails, and the nightly drift sweep fails for that stack every night and takes the shared heartbeat with it. That detection needs a credential, a workspace and a network; the same fact is a static read that belongs in `.github/tests`. The only real work is a small parser for `variable` blocks and `terraform.tfvars` assignments without HCL machinery the suite lacks; `test_a_second_environment.py` already reads `terraform.tfvars`. Both stacks assign every required variable today.

- **Kind**: A static assertion with no delta or decision.
- **Recorded by**: `add-a-staging-environment`'s code review (2026-09-10), which met the gap as that change's own in-flight state (`server_type` unassigned in `main-staging`); a check failing the tree being assembled belonged to a later change.
