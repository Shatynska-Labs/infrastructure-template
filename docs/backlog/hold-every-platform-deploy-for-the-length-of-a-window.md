---
kind: change
waits_on: []
design_with: [notice-a-pending-reboot-and-decide-what-does-it]
recorded_by: unknown
re_checked: 2026-10-19
---

`platform/README.md`'s *Upgrading the PostgreSQL major version* asks the operator to hold every other `platform/**` merge and every `platform-deploy.yml` dispatch for the length of the window. Nothing enforces that, and the failure is silent. Step 2 leaves the host without the `postgres` container its Compose definition declares. Any platform deploy that lands in the gap, whether an unrelated merge or the rebuild dispatch in `docs/runbook-rebuild.md`, recreates the container on the old major with an empty cluster, which undoes step 2 without an error. The window's own pull request is already gated, because it is opened as a draft and marked ready only after the store is cleared. A draft gates only itself.

The class is wider than PostgreSQL. The platform stack is deployed wholesale on every merge, so a deploy can undo any window that leaves the host deliberately out of step with the committed definition. `platform-deploy.yml`'s per-stack `concurrency` group serialises deploys but does not refuse one. A refusal needs state that outlives a run: a repository variable, an issue label, a file on the host, or an Environment gate. The host already carries an operator-declared window file, `/var/lib/platform-maintenance/shared-postgres-window`, which `platform-deploy.yml` does not read today. The design must cover a window abandoned half-way and decide what a held merge does: queue and land unwatched at release, or fail loudly and be re-run.

- **Kind**: It needs operator decisions on where the lock lives, who releases it, and whether a held deploy queues or fails.
- **Design with**: `notice-a-pending-reboot-and-decide-what-does-it`. An automatic-reboot policy is a scheduled outage, and this mechanism is what would keep a deploy out of it.
- **Recorded by**: `unknown`. It was recorded on 2026-09-15 from the work that gated the window's own pull request with a draft. That half was a fix, while this half needs a mechanism and decisions.
