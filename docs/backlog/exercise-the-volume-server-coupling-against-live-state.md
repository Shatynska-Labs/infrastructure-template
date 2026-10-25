---
kind: fix
waits_on: []
design_with: []
recorded_by: make-openspec-validation-a-usable-gate
re_checked: 2026-10-19
---

`terraform/stacks/main-production/main.tf` couples the volume to the server with `count = var.volume_enabled && var.server_enabled ? 1 : 0`. *Conditional Prod Volume Creation* (`openspec/specs/iac-data-volumes/spec.md`) states two scenarios against it: *Volume toggle disabled creates nothing* and *Disabling the server also removes the volume*. Nothing has ever run either against production's live state, and the module tests under `terraform/modules/volume/tests/` cannot reach a stack's `count`.

Staging's half was observed on 2026-09-16 by `write-and-rehearse-the-rebuild-runbook`. With `server_enabled = false`, its pull request's plan showed `0 to add, 0 to change, 3 to destroy` (server, firewall and volume), and the apply destroyed the volume first. That rehearsal also settled where the reads can run: the pull request's own plan job. A local plan cannot run, because every stack's `cloud { organization = "acme" }` needs an HCP credential the workstation lacks. What remains is production, where no backup covers the volume. It takes two plan-only reads, each setting one toggle to false, on a pull request that is read and then closed. That pull request is **never merged or applied**.

- **Kind**: It owes no delta and needs no decision: it runs two existing scenarios as plans and records the result.
- **Recorded by**: `make-openspec-validation-a-usable-gate`, which recovered it from `add-prod-data-volume`'s unticked task 3.5. That task sat inside an archived record, where it would otherwise have been lost.
