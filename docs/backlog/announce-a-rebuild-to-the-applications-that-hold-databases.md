---
kind: change
waits_on: []
waits_for: "a second application holding a database in the shared instance, or a rebuild performed by someone other than the operator who would announce it. Today only `orders-ops` holds one."
design_with: []
re_checked: 2026-10-19
---

*The Host Carries an Operator-Declared Maintenance Window for the Shared Instance* (`openspec/specs/iac-host-configuration/spec.md`) lets each application's probe answer `window-open` from a declaration file under `/var/lib/platform-maintenance/` during a destructive act on the shared instance. It cannot announce a rebuild, because the declaration lives on the host the rebuild destroys: the probe gets no answer, which is exactly the state of a host that has fallen over, the thing the probe exists to tell apart.

The change owes a decision on who the audience is, then where such a declaration lives: off the host (a repository variable, status page, or field the application's deploy reads) survives the rebuild but needs a second mechanism; on the host but restored early closes only part of the window; out of band to operators is not machine-readable. Today `docs/runbook-rebuild.md` phase 1 tells the operator to tell application owners by hand, and points here.

- **Kind**: It needs the operator's decision on audience and location, and owes an `iac-host-configuration` delta.
- **Recorded by**: planning `write-and-rehearse-the-rebuild-runbook` (2026-09-16), which found the gap; it was not a blocker on the runbook because a rebuild is operator-initiated and can be announced by hand.
