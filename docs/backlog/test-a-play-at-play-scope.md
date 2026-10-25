---
kind: change
waits_on: []
design_with: []
recorded_by: configure-the-staging-host
re_checked: 2026-10-19
---

`ansible/playbooks/host-baseline.yml` has a guard play that refuses when the target environment resolves to no host. Nothing tests it:

- Molecule's subject is a role on a host.
- `.github/tests` reads only committed files, and `TestEveryModuleInTheSuiteDirectoryNeedsNoPrivilegedResource` would fail a module that ran `ansible-playbook`.

So a green pull request shows only that the guard is present. One bypass is known and cannot be closed from the play: `--limit` filters `localhost` out of the guard play, so `--limit <host>` against an empty group exits 0. The play's header and `docs/bootstrap-a-new-host.md` state this bypass.

A play-scope harness would assert the refusal under each way of invoking the playbook. It would also cover other play-level behaviour: role ordering, `when:` conditions on role inclusion, and the deferred play-scope input validation noted above the playbook's role list.

- **Kind**: It adds a fourth test command (a row in `AGENTS.md`'s test-command table) and a runner for it, which is an operator decision.
- **Recorded by**: `configure-the-staging-host` (`design.md` Decision 10). That change's diff needed close reading and was no place to invent a test tier.
