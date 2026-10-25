---
kind: fix
waits_on: []
design_with: []
recorded_by: prune-unreferenced-host-images-periodically
re_checked: 2026-10-19
---

Two guards in the `image_prune` role have no assertion covering them:

- Local images are enumerated before the keep set is computed.
- Each tag is re-resolved immediately before removal.

Both are observable only when the host's images change during a run, and a black-box Molecule scenario has no point at which to change them. These two guards close the concurrent-deploy window against `app-deploy`, the only actor competing with the prune. The two abandon branches added by that change's review fix are also unasserted.

That change's code review built a rig: a stubbed `docker` on `PATH` that answers some calls and fails others. The rig showed both guards present and mutation-visible. That change's `test-plan.md` invites adding a deterministic arrangement ("sized rather than slept") to `tasks.md` 2.7 and 2.8. Nothing under `ansible/roles/image_prune/molecule/` uses such a stub yet.

- **Kind**: It adds tests for behaviour that is already specified, with no delta and no decision.
- **Recorded by**: `prune-unreferenced-host-images-periodically`. Its code review built the rig to verify the change, not to ship it.
