---
kind: change
waits_on: []
design_with: [manage-root-authorized-keys-from-a-role]
recorded_by: onboard-orders-ops-to-staging
re_checked: 2026-10-19
---

`deploy_user` renders three things per `deploy_apps` entry (`/opt/<name>`, `/etc/sudoers.d/app-deploy-<name>`, and an `authorized_keys` line carrying the forced command) and removes none of them when an entry is deleted. The `ansible.posix.authorized_key` task leaves `exclusive` at its default deliberately, so each application manages only its own key; nothing enumerates `sudoers.d` or reaps `/opt`. Deleting an entry therefore stops the role acting on it and leaves the host authorising that key exactly as before; revocation is an edit on the host or a key rotation in the application's repository, neither of which a converge notices.

Follow `ops_user`'s per-entry `state:` model (`state: absent`, entry left in place until a converge has removed it) rather than switching to `exclusive: true`, which is the only form that revokes wholesale and the form that can lock out every application. Nothing has needed revoking yet.

- **Kind**: It owes an `iac-host-configuration` delta to the multi-application deploy requirement for a revocation path.
- **Recorded by**: `onboard-orders-ops-to-staging` (2026-09-13), which added a `deploy_apps` entry and found, stating what reverting it would cost, that nothing takes one back; revocation was outside its onboarding scope.
