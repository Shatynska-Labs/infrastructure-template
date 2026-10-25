---
kind: change
waits_on: []
design_with: [catch-up-the-drifted-galaxy-pins]
re_checked: 2026-10-19
---

`.github/dependabot.yml` watches `terraform`, `github-actions`, `docker-compose` and `npm`. Nothing watches the pip manifests: `.github/requirements-ci.txt`, `ansible/requirements-test.txt` and `ansible/requirements.txt`. `ansible/requirements.txt`'s `ansible-core` must equal `requirements-test.txt`'s and `.github/tests` fails on a difference, so a bot opening a pull request against one would be red on arrival: the two must move as a pair, by a grouped update or by watching one and asserting the other follows. Do not inherit `cover-platform-images-with-dependabot`'s "one stanza" estimate: *Automated Dependency Updates* (`openspec/specs/iac-safety-hardening/spec.md`) enumerates ecosystems by name and the suite reads that back, so a new ecosystem is a stanza plus a delta plus its tests.

The watcher question is settled with a recommendation: stay with Dependabot for now. Renovate covers `pre-commit` and `ansible-galaxy` natively, but hosted Renovate is a third party with write access; self-hosted Renovate could reuse the existing autoupdate GitHub App. Revisit when a second Compose stack appears, the galaxy manifest grows past a handful of entries, or the galaxy migration is done and small bumps start being missed. The old precondition, that something notices a red scheduled run, is met by `notice-when-a-periodic-job-stops-reporting`'s heartbeat checks.

- **Kind**: It owes a delta widening *Automated Dependency Updates*' ecosystem enumeration, and a decision on how the two `ansible-core` pins move together.
- **Recorded by**: unknown change; it is the same shape as `cover-platform-images-with-dependabot`, which closed the image half, and the watcher question was recorded separately on 2026-09-08.
