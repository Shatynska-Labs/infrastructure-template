---
kind: change
waits_on: []
design_with: [make-a-rotated-secret-reach-its-inline-config]
recorded_by: apply-shipped-config-on-deploy
re_checked: 2026-10-19
---

Nothing establishes that a deploy reporting success applied everything it shipped. The proposed check runs after `docker compose up -d --wait`: compare each running container's `com.docker.compose.config-hash` with `docker compose config --hash='*'` and fail on a mismatch. That catches every container that failed to be replaced, whatever the reason. It is a backstop and does not replace the `platform.config-checksum` label: without the label, a configuration-only change hashes equal on both sides.

The check must run on the host, where `.env` is. Several platform services interpolate `${...}` from that file, so a hash computed anywhere else mismatches as an artefact of where it ran. The check edits `app-deploy` (`ansible/roles/deploy_user`), which is generic across applications. It also needs a decision on what a mismatch does to a deploy that has already replaced some services and reported them healthy.

It does not catch a secret rotated inside an embedded config, because both sides hash the same unchanged value. That case is `make-a-rotated-secret-reach-its-inline-config`. A digest taken on the host after interpolation would settle both cases. `apply-shipped-config-on-deploy` considered that deploy-time computation and rejected it in its Decision 1, so the two entries are one decision about the mechanism.

- **Kind**: It changes `app-deploy` for every application, needs the mismatch-handling decision, and owes an `iac-platform-deploy-pipeline` delta to *A Shipped Configuration Change Is Visible to the Container Runtime*.
- **Recorded by**: `apply-shipped-config-on-deploy`, which deliberately stops at making a change visible. Checking that the change was applied edits the generic `app-deploy` and needs a decision of its own.
