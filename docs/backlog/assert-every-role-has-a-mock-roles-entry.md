---
kind: fix
waits_on: []
design_with: []
recorded_by: bound-host-log-growth-and-add-swap
re_checked: 2026-10-19
---

`.ansible-lint`'s `mock_roles:` hand-lists every role referenced by name, because ansible-lint does not resolve role references through `ansible/ansible.cfg`'s `roles_path`. Nothing asserts the list is complete. A role added under `ansible/roles/` and referenced from `ansible/playbooks/host-baseline.yml` without an entry fails ansible-lint twice with a false-positive "role not found" naming a search path, which reads as a configuration problem rather than a missing line (observed 2026-09-08 adding `swap`). The list is complete today.

The check is a static read for `.github/tests`: the directories under `ansible/roles/` minus the gitignored external role, against `mock_roles`. It has the same shape as the existing "every lockfile-bearing directory is covered" check.

- **Kind**: A static assertion with no delta or decision.
- **Recorded by**: `bound-host-log-growth-and-add-swap`, which added its own missing entry; asserting the invariant was a different concern and the suite was not that change's subject.
