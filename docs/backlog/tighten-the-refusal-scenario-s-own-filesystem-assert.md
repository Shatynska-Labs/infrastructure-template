---
kind: fix
waits_on: []
design_with: []
recorded_by: namespace-the-molecule-loop-devices
re_checked: 2026-10-19
---

`ansible/roles/platform_data_volume/molecule/superseded-path-in-force-refused/verify.yml` asserts `platform_data_volume_blkid.rc != 0` to show that the role refused before formatting the device. This assertion fails open. `blkid` also returns 1 on a usage error, so a probe that never happened reads as a refusal. Only rc 2 means "looked and found nothing", and the same role's `prepare.yml` already uses `rc != 2`.

A second hazard in the same assertion remains after that fix. `prepare.yml` and `verify.yml` each redeclare `platform_data_volume_fixture_loop_minor` (`base + 4`) by hand. If `verify` drifts, it probes an unassociated minor, gets rc 2 and passes. Either derive the minor from one place, or have `verify` assert that its device is the one `prepare` associated. Fix the rc-1 case, and do not treat the drift as closed by it.

- **Kind**: It is a test-only tightening that owes no delta and needs no decision.
- **Recorded by**: `namespace-the-molecule-loop-devices`. Its code review found the problem in a line that change did not author, and it was left out of the diff to avoid another review round.
