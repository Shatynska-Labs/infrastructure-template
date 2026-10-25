---
kind: fix
waits_on: []
design_with: [share-the-static-suites-test-arrangement]
recorded_by: alert-on-certificate-expiry
re_checked: 2026-10-19
---

The self-check half is done: `TestEveryModuleInTheSuiteDirectoryNeedsNoPrivilegedResource` in `.github/tests/test_ci_configuration.py` now holds every module in `.github/tests` to the no-network, pinned-imports-only, no-container and no-Terraform constraints, beside the older `TestTheSuiteNeedsNoPrivilegedResource`, which still reads only its own file (`SUITE_PATH`). What remains is the half absorbed from `prune-the-repository-walkers-of-provisioned-content`: repository-scope walkers read a directory rather than the committed tree.

`compose_shaped_files()` (behind `TestEveryComposeFileDeclaringAServiceImageIsCovered`) walks through `walked_files()`, whose prune lists (`PRUNED_ANYWHERE`, `PRUNED_AT_ROOT_RELATIVE`) do not include `.molecule-home/`. On a working tree provisioned for Molecule it finds the `community.docker`/`community.general` fixture Compose files there and fails; CI never provisions Molecule and stays green. It is also a race while Molecule creates and removes `tmp/` under that directory (observed 2026-09-11). The fix is to point repository-scope walkers at the existing `tracked_files()` helper in the same module, which closes the class (also `.terraform/`, virtual environments, sibling working trees), not to prune one more directory. **Must not undo**: deleting `.molecule-home/` to get green throws away the provisioning the Molecule tier depends on.

- **Kind**: No delta and no decision: the helper and its admitted `git` use already exist.
- **Design with**: `share-the-static-suites-test-arrangement`, which gives the same suite a support module.
- **Recorded by**: `alert-on-certificate-expiry` (the self-check half, as the change that added a second suite module), and the test author deriving `cache-the-apt-index-within-a-converge` (the walker half, met as a red suite); neither change's subject was the suite's walkers.
