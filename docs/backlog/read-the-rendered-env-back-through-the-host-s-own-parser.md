---
kind: change
waits_on: []
design_with: []
recorded_by: render-the-env-file-so-a-secret-survives-it
re_checked: 2026-10-19
---

`platform-deploy.yml` escapes each value it writes into `platform/.env` (double quotes, then `\` → `\\`, `"` → `\"`, `$` → `$$`, in that order) so that Compose's dotenv parser reconstructs it. That rule belongs to one version of the parser: it was measured on Compose v5.4.0 while the host ran v5.5.0, and both versions move under Dependabot and host package updates. A parser that diverges does not fail the deploy. The service starts, reports healthy, and fails wherever it needed the credential. `.github/tests/test_the_rendered_env_survives_its_parser.py` checks only that the escaping exists and runs in order.

The fix is a round trip at deploy time, on the host: render the file, parse it back with the host's own Compose, compare digests rather than values so that nothing is disclosed, and refuse the deploy on any mismatch. `tools/env-rendering-probe/` already holds the harness, both consumption paths and the value corpus.

- **Kind**: It adds a deploy-time refusal, which owes an `iac-platform-deploy-pipeline` delta to *Platform Secrets Rendered from CI at Deploy Time*.
- **Recorded by**: `render-the-env-file-so-a-secret-survives-it`. That change measured the rule but could not arrange for a divergence to be noticed, and the check belongs to the deploy path.
