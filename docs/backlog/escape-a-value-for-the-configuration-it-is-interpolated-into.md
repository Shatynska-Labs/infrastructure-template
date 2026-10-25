---
kind: change
waits_on: []
design_with: [make-a-rotated-secret-reach-its-inline-config]
recorded_by: render-the-env-file-so-a-secret-survives-it
re_checked: 2026-10-19
---

The `.env` escaping guarantees only what the dotenv parser yields. `SLACK_WEBHOOK_URL` and `DEADMANSWITCH_URL` are then interpolated unquoted into the embedded Alertmanager YAML in `platform/docker-compose.yml`, and `ACME_EMAIL` goes into a Traefik `command:` argument. A `: `, a `#`, a quote or a line break in one of these values would change or break the document it lands in, and nothing escapes for either context. Today's vendor-issued values carry no such character, but that is a property of the values and not of the file, so this work is still owed. `platform/README.md` states the residual.

One decision settles it: escape at render time, with a second escaper per destination, or move the two URLs out of the embedded configuration through Alertmanager's file reference, which removes the question. Either way, a corrected value reaches the container only when the container is next replaced (*A Shipped Configuration Change Is Visible to the Container Runtime*, `openspec/specs/iac-platform-deploy-pipeline/spec.md`).

- **Kind**: It needs the escape-or-file-reference decision and owes an `iac-platform-deploy-pipeline` delta.
- **Design with**: `make-a-rotated-secret-reach-its-inline-config`, because moving the URLs out of the embedded config changes how a rotation reaches the container.
- **Recorded by**: `render-the-env-file-so-a-secret-survives-it`, whose obligation deliberately stops at what the dotenv parser yields.
