---
kind: change
waits_on: []
design_with: []
re_checked: 2026-10-19
---

`apply.yml` names its failures and the recovery for most cases, but a stale saved plan only gets Terraform's raw `Error: Saved plan is stale`. The refusal itself is the gate working: *Gated Production Apply Applies the Reviewed Plan* demands the exact plan a human reviewed. What the change owes is a named failure for this case. It should say what staleness means, that no infrastructure change was lost, and that recovery is a **fresh run of the whole workflow**, because re-running the failed job re-downloads the same stale artifact. It must not re-plan automatically, which would apply a plan nobody reviewed.

The case is likely because production's apply waits for a required reviewer, and that wait is the window in which the plan goes stale. It was observed on 2026-09-12, on the merge of `rename-the-external-services` (PR #155). Production's state serial moved from 44 to 45 between plan and apply, with no attribute changed, which shows that even a no-op operation bumps the serial. A local `terraform plan` under the read-only token was measured not to bump it. CI, the operator's checkout, the change's working trees and HCP health assessments were all ruled out as the cause. The one avenue not yet read is the workspace's **States** view in HCP, which labels each state version with its source.

- **Kind**: It needs a decision on whether the message names likely causes. If a permitted local operation turns out to invalidate a pending apply, it also needs an `AGENTS.md` rule change.
- **Recorded by**: the merge of `rename-the-external-services` (PR #155), whose production apply failed this way. The error-handling gap lay outside that change's subject.
