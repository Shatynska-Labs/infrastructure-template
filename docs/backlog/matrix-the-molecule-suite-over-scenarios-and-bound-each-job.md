---
kind: change
waits_on: []
design_with: []
recorded_by: promote-molecule-to-a-required-check
re_checked: 2026-10-19
---

`molecule test --all` stops at a role's first failing scenario and leaves the remaining scenarios out of the SCENARIO RECAP, so a red run under-reports while looking complete. Molecule's own remedy, `--continue-on-failure`, needs `--workers`, and `--workers` refuses outside collection mode (observed 2026-09-07). The remedy that works is an `ansible-verify.yml` matrix over **scenarios** rather than roles; today the job is `molecule (${{ matrix.role }})`. It also parallelises the longest role. This is not the shared-state hazard, which per-working-tree namespacing has already closed.

The same change bounds each job. No workflow sets `timeout-minutes`, so every job inherits GitHub's 360-minute default. Because `ansible-verify` is a required check, a hung scenario leaves a pull request pending for six hours, and `AGENTS.md` describes ways a Molecule run hangs. The timeout must sit above the slowest run that still passes and below six hours. Choose it against durations measured after `cache-the-apt-index-within-a-converge`, and record them in the proposal.

It must not undo two requirements in `openspec/specs/iac-cicd-pipeline/spec.md`:

- *Required Status Checks Report on Every Pull Request*: the aggregating job with a literal name stays as the registered check.
- *Ansible Configuration Is Verified in Continuous Integration and Gates the Merge*: scenarios are discovered from `ansible/roles/*/molecule/*/` without a workflow edit, and an empty result fails loudly.

- **Kind**: It reshapes the jobs behind a required check and needs an operator decision on the timeout value.
- **Recorded by**: `promote-molecule-to-a-required-check`, which named it a non-goal so that the change registering the check did not also redefine it. The timeout half was recorded on 2026-09-11, alongside `cache-the-apt-index-within-a-converge`.
