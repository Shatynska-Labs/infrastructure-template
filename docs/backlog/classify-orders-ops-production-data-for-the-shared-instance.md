---
kind: change
waits_on: []
design_with: [move-orders-ops-durable-data-to-examplepg]
recorded_by: provision-orders-ops-database-in-the-shared-instance
re_checked: 2026-10-19
---

On 2026-09-14 the operator decided that production's `orders-ops` data leaves its private PostgreSQL for the database already provisioned in production's shared instance, and moves to ExamplePG only later. That data includes hand-curated rows, and the operator classified it as tolerable to lose, to the operator, with no backup. The policy is temporary, taken because the application is at an early, experimental stage, and it ends with the move to ExamplePG. *Single Shared PostgreSQL Instance, Per-Application Databases* (`openspec/specs/iac-platform-services/spec.md`) still keeps durable data out of the shared instance unconditionally, so the specification does not yet record this decision.

The change owes a MODIFIED delta on that requirement, shaped like its staging rehearsal-data policy: whose loss is tolerable, which application and host the policy covers, and that it ends with the move to ExamplePG. It also owes a delta on *No Store on This Host Holds Data Requiring Backup* (`openspec/specs/iac-safety-hardening/spec.md`). It must merge before any production row lands in the shared instance, and the `orders-ops` repository's production cutover waits on it. `platform/README.md`'s major-upgrade section already warns that once the data moves in, clearing production's store deletes it for good. `docs/runbook-rebuild.md` does not carry that warning yet, and its procedure needs a re-read once this change lands.

It must not undo the following: the classification names one application on one host and gives no permission for any other application's durable data.

- **Kind**: It owes MODIFIED deltas on *Single Shared PostgreSQL Instance, Per-Application Databases* and *No Store on This Host Holds Data Requiring Backup*.
- **Design with**: `move-orders-ops-durable-data-to-examplepg`, whose step 2 is this change.
- **Recorded by**: the operator's decision of 2026-09-14, taken after `provision-orders-ops-database-in-the-shared-instance` had merged, so that change could not carry it.
