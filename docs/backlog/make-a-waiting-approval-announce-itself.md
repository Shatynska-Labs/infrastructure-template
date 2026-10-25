---
kind: change
waits_on: []
design_with: []
recorded_by: rename-the-external-services
re_checked: 2026-10-19
---

Nothing tells the operator that a gated run is waiting for approval. On 2026-09-12 a production converge sat unapproved from 06:26 to 09:56, and on 2026-09-13 another waited 6h34m. Each was found only because a later run queued behind it with `waiting on converge (main-production) … to complete`, which reads like a hung job. The Terraform apply, the host converge and the platform deploy gate on the same Environment and render prompts that cannot be told apart. A pending approval also stalls every later converge of that stack.

Before building anything, measure whether GitHub's *deployment review requested* notification exists and simply does not reach the operator. If it does, the fix is a settings change plus a sentence in `docs/bootstrap-a-new-host.md` §6.6. Otherwise, build a scheduled query of `gh run list --status waiting` that pushes to somewhere the operator reads, and repeats while the state persists. There is a trap: Alertmanager's `SLACK_WEBHOOK_URL` comes from `PLATFORM_SLACK_WEBHOOK_URL`, a production-Environment secret. A notifier using that secret would itself wait on the approval it is meant to announce, so its credential must come from somewhere ungated. It must not fire on every run.

- **Kind**: It needs operator decisions on the channel and the credential source, and may add a scheduled workflow, which carries a heartbeat obligation.
- **Recorded by**: `rename-the-external-services`, which lost three and a half hours to the problem and recorded only advice on recognising it.
