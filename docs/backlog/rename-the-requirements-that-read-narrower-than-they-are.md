---
kind: change
waits_on: []
design_with: [sweep-the-stale-scenario-titles-and-check-them]
recorded_by: make-the-pipeline-environment-agnostic
re_checked: 2026-10-19
---

Requirement names are this repository's citation form, and several still name production or "this host" alone although two stacks now share them. Left by `make-the-pipeline-environment-agnostic`: *Gated Production Apply Applies the Reviewed Plan* (a genuine name/content mismatch) and *Credential Scoping by Privilege* (`iac-cicd-pipeline`), *Write Credentials Confined to the Gated Pipeline* (`iac-safety-hardening`). A second set: *Conditional Prod Server Creation* (`iac-server-lifecycle`), *Conditional Prod Volume Creation* (`iac-data-volumes`), *Data Durability for Stateful Resources* and *No Store on This Host Holds Data Requiring Backup* (`iac-safety-hardening`), *Host Joins a Private Tailnet for Non-Operator SSH Access* and *Unprivileged Operator Accounts Support Interactive Host Inspection* (`iac-host-configuration`). Staging uses the same toggles, tailnet and operator account, and staging took the platform stack on 2026-09-13 and an `orders-ops` database the next day, so it holds stores: `platform_postgres_data`, `platform_traefik_letsencrypt` and Alertmanager's volume; "this host" is now wrong about which host a durability obligation binds.

Renaming sweeps every citation: comments in `apply.yml`, `drift.yml` and `pr-validation.yml`, `docs/bootstrap-a-new-host.md`, `.github/tests`, and archived records. `.github/tests/test_a_retired_requirement_name_is_reported.py` makes the sweep checkable. Precedent: `add-a-staging-environment` renamed *Each Stack Has a Dedicated Hetzner Cloud Project* where it had a content change to carry it.

- **Kind**: It owes rename deltas across four capabilities and a repository-wide citation sweep.
- **Design with**: `sweep-the-stale-scenario-titles-and-check-them`, the same citation-sweep mechanism over scenario titles.
- **Recorded by**: `make-the-pipeline-environment-agnostic` (first set) and the staging work that followed (second set); both declined to mix a mechanical rename into diffs that most needed reading closely.
