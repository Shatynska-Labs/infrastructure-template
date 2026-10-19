# Backlog

Changes this project has identified and not yet archived. Each entry is a change recorded here instead of being folded into the change that found it, or opened on a branch of its own.

An entry's heading is the change's name in kebab-case — the name its branch and change directory are expected to take, though the session that opens the change may rename it if the work turns out different. A fix takes the name as its branch's only. Refer to an entry only by that name.

The order of entries carries no meaning. What an entry waits on, and which entries share a design with it, it states in its own fields.

Keep an entry short: what is needed and why, naming files, specifications and changes as pointers. Leave out line numbers, census tables and step-by-step plans; they go stale first, and the session that takes the entry up redoes them.

Delete an entry when its change is archived, when the operator takes it as a fix rather than a change, when a change makes it obsolete, or when the operator decides the change will not be made. When a change is archived, also update the entries it unblocked or made stale, and set their **Re-checked** date. Keep this file when it has no entries.

An entry takes this shape:

```
## `change-name`

What the change is and why it is needed, written for a session that has not seen the work that found it.

**Kind**: `fix` or `change`, and why. A fix owes no specification delta, needs no decision from the operator, and fits one review sitting; anything else is a change, and says which delta it owes, which decision it needs, or that it is too large.

**Waits on**: `nothing`, or something that can be checked: a change by name, a person's answer, an event.

**Design with**: the entries that share a design with this one, by name, or `nothing`.

**Recorded by**: the change that identified it, and why it was not done there.

**Re-checked**: the date the entry was last verified against the repository.
```

## `close-public-ssh-and-manage-sshd-explicitly`

Port 22 is open on the cloud firewall and in UFW to one `/24` (`ssh_allowed_cidrs` in each stack's `terraform.tfvars`, mirrored as `hardening_ssh_allowed_cidrs` in `group_vars`). Every non-operator path, `host-converge.yml` included, already reaches hosts over the tailnet, which `ansible/roles/hardening/tasks/main.yml` admits independently from `100.64.0.0/10` (its own comment calls an empty public list "safer than what prod runs"), so the public rule serves only the operator, who is on the tailnet too. Closing it means `ssh_allowed_cidrs = []`, which `terraform/modules/server/variables.tf` refuses on lockout grounds; that validation is what to revisit. The Terraform and UFW rules move together (*Host-Level Security Owned by Ansible, Cloud Firewall Owned by Terraform*), and the recovery path when the tailnet is unreachable is Hetzner's console, which the cloud firewall does not gate.

Must not break: the public address is the only path for a host's *first* converge (`docs/bootstrap-a-new-host.md` §6.3, `docs/runbook-rebuild.md`'s converge phase), since that play is what joins the tailnet. The change must say how a new or rebuilt host is bootstrapped afterwards (a temporarily widened `ssh_allowed_cidrs` through the pipeline, or the console). It must not precede a converge pipeline seen to work over more than one production run; the public `/24` is the fallback for a converge that wedges `tailscaled`.

Also here: `sshd_config` (no drop-in under `/etc/ssh/sshd_config.d/`; `PasswordAuthentication` unset, harmless only because no account has a password) runs on the image's defaults although the requirement above assigns SSH hardening to Ansible. It belongs to `hardening` and can be asserted by its Molecule scenario. The `unattended-upgrades` reboot half belongs to `notice-a-pending-reboot-and-decide-what-does-it`.

**Kind**: `change`. It needs the operator's policy decision to close public SSH and to choose the bootstrap path, and owes deltas to the server module's lockout validation (`iac-safety-hardening` / `iac-server-lifecycle`) and to `iac-host-configuration` for a managed `sshd_config`.

**Waits on**: `nothing`. It used to wait on the host converge moving into the pipeline, which `host-converge.yml` now does over the tailnet.

**Design with**: `notice-a-pending-reboot-and-decide-what-does-it`.

**Recorded by**: `unknown`; the entry predates the tailnet-based converge and was kept as a policy decision the `hardening` role anticipates.

**Re-checked**: 2026-10-19.

## `notice-a-pending-reboot-and-decide-what-does-it`

Both hosts carried `/var/run/reboot-required` (kernel, `linux-base`, `libc6`) from 2026-09-10 to 2026-09-16 with nothing reporting it, found only when they were rebooted by hand. `unattended-upgrades` is installed and enabled by the image, not by `hardening`, and every `Automatic-Reboot` line in `/etc/apt/apt.conf.d/` is commented out, so patches install and do not take effect for an unbounded time while the host looks healthy.

It is two changes under one name. *Noticing* is a metric and an alert: node-exporter's textfile collector is not enabled (`platform/docker-compose.yml`'s `node-exporter` has no `--collector.textfile.directory` and no mount for one), so it needs a `platform/` change for the collector and rule plus an `ansible/` unit that writes the metric, reaching the host through two pipelines. *Deciding what reboots* is a `hardening` change: an `Automatic-Reboot` policy with a window, or a stated operator ritual. Rebooting is known-safe: both hosts rebooted cleanly on 2026-09-16 in about forty seconds (containers back on `restart: unless-stopped`, `/mnt/main` mounted before Docker, swap, `ufw`, `tailscaled` up). A reboot is an outage of every application on the host, so an automatic production policy is a scheduled outage; `hold-every-platform-deploy-for-the-length-of-a-window` is what would keep a deploy out of it.

**Kind**: `change`. It needs the operator's reboot-policy decision, spans two pipelines, and owes deltas to `iac-host-configuration` (unattended upgrades owned by `hardening`) and `iac-platform-services` (the collector and alert).

**Waits on**: `nothing`.

**Design with**: `alert-on-a-scheduled-units-own-output` (the same unenabled textfile collector: flag, directory, ownership and mount built once for both), `close-public-ssh-and-manage-sshd-explicitly` (the other half of the image-default `hardening` gap), `hold-every-platform-deploy-for-the-length-of-a-window`.

**Recorded by**: the operator and a session on 2026-09-16, from the hosts after rebooting them by hand; not by a change.

**Re-checked**: 2026-10-19.

## `manage-root-authorized-keys-from-a-role`

No role manages `root`'s `authorized_keys`: its first entry comes from Hetzner at server creation (`hcloud_server.ssh_keys` cannot change without recreating the server), and each stack's `ansible-ci` converge key is appended by hand at `docs/bootstrap-a-new-host.md` §6.6. Revoking the most powerful credential CI holds is therefore an edit on the host rather than a commit.

A role owning that file can lock every operator out, a different class of risk from anything `hardening` does, so it wants Molecule scenarios of its own. Follow `ops_user`'s per-entry `state:` model (revocation is `state: absent`, the entry left in place until a converge has removed it). Only `exclusive: true` actually revokes, and that is the form that can lock everyone out. The first installation stays manual whatever the shape; the change buys every one after.

**Kind**: `change`. It owes an `iac-host-configuration` requirement for a role owning `root`'s keys, and carries lockout risk too large for a fold-in.

**Waits on**: `nothing`.

**Design with**: `revoke-an-application-s-deploy-authorisation` (same revocation shape and the same `exclusive` trade-off).

**Recorded by**: `apply-host-configuration-through-a-gated-workflow`, which installs the converge key and cannot revoke it; not done there because of the lockout risk.

**Re-checked**: 2026-10-19.

## `revoke-an-application-s-deploy-authorisation`

`deploy_user` renders three things per `deploy_apps` entry (`/opt/<name>`, `/etc/sudoers.d/app-deploy-<name>`, and an `authorized_keys` line carrying the forced command) and removes none of them when an entry is deleted. The `ansible.posix.authorized_key` task leaves `exclusive` at its default deliberately, so each application manages only its own key; nothing enumerates `sudoers.d` or reaps `/opt`. Deleting an entry therefore stops the role acting on it and leaves the host authorising that key exactly as before; revocation is an edit on the host or a key rotation in the application's repository, neither of which a converge notices.

Follow `ops_user`'s per-entry `state:` model (`state: absent`, entry left in place until a converge has removed it) rather than switching to `exclusive: true`, which is the only form that revokes wholesale and the form that can lock out every application. Nothing has needed revoking yet.

**Kind**: `change`. It owes an `iac-host-configuration` delta to the multi-application deploy requirement for a revocation path.

**Waits on**: `nothing`.

**Design with**: `manage-root-authorized-keys-from-a-role`.

**Recorded by**: `onboard-orders-ops-to-staging` (2026-09-13), which added a `deploy_apps` entry and found, stating what reverting it would cost, that nothing takes one back; revocation was outside its onboarding scope.

**Re-checked**: 2026-10-19.

## `say-what-the-host-firewall-actually-gates`

A container-published port is not filtered by UFW on any interface: Docker DNATs it in `nat/PREROUTING` and accepts it in `FORWARD`, and UFW's rules hang off `INPUT`. So `hardening_web_allowed_cidrs` gates nothing for Traefik's `0.0.0.0:80/443`, and the Hetzner cloud firewall alone refuses public traffic (measured on staging 2026-09-13 with UFW active and both CIDR lists empty: 301 and 404 over the tailnet). Nothing in `ansible/roles/` touches `DOCKER-USER` or `after.rules`.

This falsifies the two-layer statement where it is still made unqualified: `AGENTS.md`'s *Firewall split*, the requirement *Host-Level Security Owned by Ansible, Cloud Firewall Owned by Terraform* ("exactly one layer SHALL be the documented access gate", `openspec/specs/iac-host-configuration/spec.md`), and `ansible/roles/hardening/README.md` (its sync and tailnet-allow paragraphs). Each must say which ports it is true of. `docs/onboard-an-application.md`, `docs/bootstrap-a-new-host.md`, `group_vars/staging.yml` and `check-public-endpoints-from-outside` already say it correctly. The `terraform.tfvars`/`group_vars` mirror obligation still holds for every port UFW does gate.

Settle first: `add-grafana-tailnet-ufw-rule` added a UFW allow for 3000 (still in `hardening/tasks/main.yml`) because Grafana was unreachable from the tailnet, yet Grafana publishes on the tailnet address, a container-published port UFW should never filter. Either that diagnosis was wrong or something distinguishes the case. This is a documentation defect, not an exposure; whether to close container ports at the host layer too (`DOCKER-USER`, or publishing to `127.0.0.1`) is a separate change this one does not prejudge.

**Kind**: `change`. It owes an `iac-host-configuration` delta to *Host-Level Security Owned by Ansible, Cloud Firewall Owned by Terraform* saying which layer gates a container-published port, and an investigation of the Grafana case before the rule is written.

**Waits on**: `nothing`.

**Design with**: `put-the-remaining-host-scoped-variables-on-the-host-axis` (its firewall-pair decision depends on which ports UFW gates).

**Recorded by**: `onboard-orders-ops-to-staging`, whose code review probed the host and caught three documents asserting the opposite; that change corrected only `group_vars/staging.yml`'s comment, which it was already editing.

**Re-checked**: 2026-10-19.

## `put-the-remaining-host-scoped-variables-on-the-host-axis`

*A Host-Scoped Variable Lives in the Host's Own Vars File* (`openspec/specs/iac-host-configuration/spec.md`) names three placements it is to be read as unmet in, and this entry resolves them with it: `hardening_ssh_allowed_cidrs` and `hardening_web_allowed_cidrs` (they mirror one stack's `terraform.tfvars`, so are per stack by nature) and `ops_user_accounts` (a root-equivalent `docker`-group login; an operator of one tenant's host is not thereby one of another's). The requirement's paragraph is replaced when they move, and not before.

Each needs a decision: for the firewall pair, whether per-stack CIDRs defeat an environment-wide hardening baseline; for the accounts, whether host access is per environment or per tenant. An audit of the rest of both `group_vars` files against the requirement's test is part of the entry; `ghcr_pull_token`/`ghcr_pull_username` come first, since two tenants' GHCR organisations differ. The mechanism (`ansible/inventory/host_vars/<server name>.yml`) already exists for both hosts and takes precedence.

Ride-along: `.github/workflows/host-converge.yml` checks that a stack's `group_vars` file exists but not its host vars file, so a `workflow_dispatch` run lacks the guard a pull request gets from `.github/tests`. Add the host-vars half beside the existing group-vars check; it completes a pair rather than restating a role's contract.

**Kind**: `change`. It needs the operator's per-environment-or-per-tenant decisions and owes an `iac-host-configuration` delta to the requirement's unmet paragraph.

**Waits on**: `nothing`.

**Design with**: `say-what-the-host-firewall-actually-gates`.

**Recorded by**: `bound-a-deploy-key-to-one-host-when-an-environment-holds-two-stacks`, which moved `deploy_apps` to host vars and left these deliberately because each needs a decision it did not make; it also declined the dispatch-path check on cost.

**Re-checked**: 2026-10-19.

## `narrow-the-converge-trigger-to-what-a-converge-reads`

`host-converge.yml` triggers on any push to `main` touching `ansible/**`, so comment-only and test-fixture changes converge both hosts (`changed=0`), three times observed: PR #148 (2026-09-12), PR #161 (2026-09-13, Molecule scenarios and `ansible/scripts/run-molecule` only, which held a production approval open 6h34m overnight), and PR #210 (2026-09-15, a comment-only `requirements.yml`/`requirements.txt` diff). A converge changes the firewall, logins and container runtime; relying on idempotency is weaker than not running.

It is not a copy of `narrow-the-molecule-trigger-to-what-it-reads`: the exclusion sets are near inverses. A converge reads the inventory, `group_vars`, `host_vars`, the playbooks and every role's `tasks/`, `defaults/`, `handlers/`, `templates/`, `meta/`, so all of those stay. The safe exclusions are roughly documentation (`**/README.md`), `.envrc*`, and Molecule scenario directories plus `ansible/scripts/run-molecule`.

Must not: let a host silently not converge. A wrong filter reports green while a host keeps its old firewall. Keep the exclusion list conservative, use the `some-with-excludes` discipline `ansible-verify.yml` documents, and add a `.github/tests` assertion that every path a converge reads is not excluded. A reviewer may legitimately keep the coarse trigger because that failure mode is silent.

**Kind**: `change`. It owes an `iac-cicd-pipeline` delta for the converge trigger and needs a decision that narrowing is worth the silent-failure risk.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: the session that merged `rename-terraform-environments-to-stacks` (PR #148) on 2026-09-12; outside that change's subject.

**Re-checked**: 2026-10-19.

## `make-the-converge-survive-a-galaxy-outage`

The Galaxy install (`ansible-galaxy collection install` then `role install`, from `ansible/requirements.yml`) runs with no retry, cache or fallback in `host-converge.yml`, `ansible-verify.yml` and `pr-validation.yml`. It failed transiently twice on 2026-09-12: a gated production converge (PR #155, a cache-shaped "Missing expected 'results'" error) and a Molecule job on PR #161 (connection reset fetching `hetzner.hcloud`). Neighbouring identical installs passed and re-runs went green. On the gated production path the cost is a spent approval and a host left unconverged.

Must not: a blind retry would hide a genuine pin failure. `ansible/requirements.yml` pins exact versions, so "could not reach the server" must be distinguished from "this version does not exist", and only the first retried. Options in rising cost: bounded retry with backoff; `--no-cache`/`--clear-response-cache`; caching resolved collections between runs (interacts with pinning); vendoring the collections. The fix belongs to every workflow that installs Galaxy content.

**Kind**: `change`. It needs a decision among the options and spans three workflows.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: the session that merged `rename-the-external-services` (PR #155), whose gated production converge failed on it; outside that change's subject.

**Re-checked**: 2026-10-19.

## `detect-host-drift-on-a-schedule`

Nothing reports what a host has drifted to between converges; the host layer is the only one without that (`drift.yml` sweeps Terraform nightly; the platform stack is redeployed on every merge). A scheduled `--check --diff` against each host is the host layer's `drift.yml`. It cannot live in the converge workflow, since a pre-approval job must hold no credential that reaches a host and check mode must authenticate.

It cannot ship on today's baseline: the `tailscale` role's two `get_url` tasks (*Add the Tailscale apt signing key*, *Add the Tailscale apt repository*) have no `checksum:` and report changed under `--check` forever, so a healthy host reads `changed=2`. Remedies: pin a checksum (upstream rotates the key), replace `get_url` with something check-mode-verifiable, or filter those two by name with the reason stated. Check mode also skips `command` tasks (`ops_user`, `swap`) and `geerlingguy.docker` ignores errors in check mode, so a clean check speaks for files and packages, not the host; the report must say so where it reports. `docs/bootstrap-a-new-host.md` §6.3 states the `changed=2` baseline and changes when this lands.

Shape it as `drift.yml`'s sibling: scheduled, per environment, one deduplicated issue per environment, and a liveness report (*Scheduled Workflows Report Their Own Liveness*).

**Kind**: `change`. It adds a scheduled workflow owing an `iac-cicd-pipeline` delta and needs a decision on the `get_url` remedy.

**Waits on**: `report-an-absent-tailscale-auth-key`, which brings the `tailscale` role its first Molecule scenario; checkable as `ansible/roles/tailscale/molecule/` existing.

**Design with**: `nothing`.

**Recorded by**: `apply-host-configuration-through-a-gated-workflow` (its `design.md` Decision 10), which made it possible and declined it because a check-mode run needs a host credential the pre-approval job must not hold.

**Re-checked**: 2026-10-19.

## `report-an-absent-tailscale-auth-key`

*A Role's Absent Required Input Is Reported by Name* (`iac-host-configuration`) is scoped to inputs consumed on every run; `tailscale_auth_key` is the excluded class. It is consumed only by *Bring the host onto the tailnet*, under a `when:` that skips an already-joined host, so an unconditional assertion would break every re-converge.

The diagnostic must fire under the same four-limb condition as the join, stated once; reading its `POLARITY` comment wrong silently stops a host joining the tailnet, which the deploy pipeline depends on. The consuming task's `no_log: true` censors an absent key into a redacted error. `host-converge.yml` passes `tailscale_auth_key=''` on every converge by design, so "undefined" and "empty" differ: firing on undefined never fires for the pipeline, and firing on empty fires on every pipeline converge. The trigger is the join condition itself. The role has no Molecule scenario; this change brings its first.

**Kind**: `change`. It owes a delta to *A Role's Absent Required Input Is Reported by Name* for a conditionally consumed input, and needs the absent-versus-empty decision.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `fix-volume-discovery-and-consistency` (its `design.md` Decision 3a), whose requirement deliberately excluded conditionally consumed inputs.

**Re-checked**: 2026-10-19.

## `collect-each-role-s-apt-installs-into-one-task`

`hardening` installs `ufw` and `fail2ban` as two `apt` tasks where `ansible.builtin.apt` takes a list. Collapsing them reaches the case `cache-the-apt-index-within-a-converge`'s staleness bound does not: a production host with a maintained `update-success-stamp`, where `cache.update()` never advances what `get_cache_mtime()` reads, so every bounded task fetches again. It saves little in CI, where the first fetch of a fresh container is owed either way.

It moves things readers depend on: which task a failure surfaces under, what the scenarios assert (they observe `ufw` and `fail2ban` separately), and the install-then-configure ordering. Measure the production saving before proposing it.

**Kind**: `change`. It needs a measured decision that the production saving justifies changing the roles' task structure and their scenario assertions.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `cache-the-apt-index-within-a-converge` (its Non-Goals); a larger change to two roles than that bound.

**Re-checked**: 2026-10-19.

## `reclaim-a-multiply-referenced-untagged-image`

An untagged image with more than one repository reference (pulled by digest from a registry and its mirror) cannot be removed by identity unforced: the runtime refuses with "referenced in multiple repositories", so the host prune offers and refuses it every run, now counted under `refused` with the reason named (`ansible/roles/image_prune/README.md`).

Must not: force removal; *Unreferenced Host Images Are Pruned on a Schedule* forbids it because the unforced refusal backstops a defective keep set. The likely shape is removing through each repository reference in turn, as tags are, but that makes a digest reference something the run drops and has no counterpart to the pre-removal re-check that guards a re-pointed tag. Neither host is known to carry such an image.

**Kind**: `change`. It changes what the run removes, owing a delta to *Unreferenced Host Images Are Pruned on a Schedule*.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `report-refused-removals-in-the-host-prune` (2026-09-15), which made the condition visible; changing removal was outside a reporting change.

**Re-checked**: 2026-10-19.

## `size-platform-container-resource-limits`

No service in `platform/docker-compose.yml` declares a memory or CPU limit on a `cx33`, so one container can starve the host; `ContainerRestartingOrOOMKilled` only alerts on the consequence. Size limits from observed steady state and peak (`container_memory_usage_bytes` by container, on the "Container health" dashboard), not by guess: limits picked without evidence cause the outage they were meant to prevent.

**Kind**: `change`. It needs sizing decisions from measured data across every platform service.

**Waits on**: the operator's judgement that the "Container health" dashboard holds enough history to show real steady state and peak; metrics have accumulated since `add-platform-monitoring`.

**Design with**: `nothing`.

**Recorded by**: `unknown`.

**Re-checked**: 2026-10-19.

## `alert-on-swap-utilisation`

Nothing alerts when swap is being consumed, the sign that a leak is underway and the OOM killer is next. `HostMemoryPressure` reads `MemAvailable / MemTotal` (RAM only), so it still fires on a leak but cannot say the last-resort tier is draining. `node_memory_SwapFree_bytes` and `node_memory_SwapTotal_bytes` are already scraped, so the rule sits beside the others in `platform/docker-compose.yml`'s inline `prometheus_rules`. Decide level (fraction used) versus rate (consumed per unit time); with no history, pick the simpler and revisit.

**Kind**: `change`. It needs a threshold decision and owes an `iac-platform-services` alerting delta.

**Waits on**: `size-platform-container-resource-limits`; container limits change what swap absorbs, so a threshold chosen first describes a host about to change.

**Design with**: `nothing`.

**Recorded by**: `bound-host-log-growth-and-add-swap`, which added swap; this is a `platform/` change on a different pipeline, so folding it in would have needed two deploys.

**Re-checked**: 2026-10-19.

## `alert-on-the-exporter-being-unable-to-read-postgres`

`MetricsTargetDown` (`up == 0`) cannot catch postgres-exporter failing to reach the database: the exporter answers `/metrics` 200 with `pg_up 0`, and its healthcheck passes (measured against the pinned `v0.20.1`). On 2026-09-15 both hosts were found at `pg_up 0` for an unknown period (a `#` in the password broke `DATA_SOURCE_NAME`, since fixed by splitting the connection settings) with nothing reporting it. The `pgexporter` role is recreated by hand after any store reset or rebuild, the step most likely to go wrong; `platform/README.md` and a comment in `platform/docker-compose.yml` still say nothing alerts on `pg_up`.

Owed: an alert on `pg_up == 0` for the `postgres-exporter` job in the inline `prometheus_rules` config (not `prometheus_config`, where a `groups:` block is not read), with a `for:` that rides out an instance restart, and the `platform.config-checksum` label regenerated or nothing deploys. The wider question is whether each exporter needs its own liveness signal rather than one alert per incident.

**Kind**: `change`. The rule is one line, but it needs the per-exporter liveness decision and owes an `iac-platform-services` alerting delta.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `unknown` change; found 2026-09-15 while writing `platform/README.md`'s *Upgrading the PostgreSQL major version*, and kept open for the wider exporter question.

**Re-checked**: 2026-10-19.

## `alert-on-a-scheduled-units-own-output`

`prune-host-images` writes its outcome to the journal and nothing reads it. The dead-man's switch covers the unit not running, not what a successful run reported, so a prune that refuses the same image weekly, or whose `considered` count collapses, is green. The sharpest case: a runtime that stops responding mid-loop leaves the prune reporting every remaining candidate refused and still exiting zero, so the heartbeat goes green on a half-done run (signature in `ansible/roles/image_prune/README.md`, which cites this entry).

Mechanism: node-exporter's textfile collector, with rules in the inline `prometheus_rules`. The collector is not enabled (no `--collector.textfile.directory`, no mount), and the first entry taken pays for the flag, directory, ownership and mount. Decide whether the prune script writes the textfile (making metrics part of a script governed by *A Run That Cannot Determine the Keep Set Completely Removes Nothing*) or a wrapper does (a second thing that can silently stop). Shape it for every scheduled host unit, not the prune alone.

**Kind**: `change`. It needs the script-or-wrapper decision, spans a role, a collector directory and alert rules, and owes deltas to `iac-host-configuration` and `iac-platform-services`.

**Waits on**: `nothing`.

**Design with**: `notice-a-pending-reboot-and-decide-what-does-it`.

**Recorded by**: `report-refused-removals-in-the-host-prune`, which found `image_prune`'s README already citing this deferral without a record; out of a reporting change's scope.

**Re-checked**: 2026-10-19.

## `label-each-stack-s-alerts-with-the-stack-they-came-from`

Prometheus in `platform/docker-compose.yml` declares no `external_labels`, and Alertmanager routes group on labels that name no host or stack, so staging's and production's alerts are identical text. Attribution currently comes from each stack's own `PLATFORM_SLACK_WEBHOOK_URL` and `PLATFORM_DEADMANSWITCH_URL` (`docs/bootstrap-a-new-host.md` §7.3), which nothing enforces: paste one stack's webhook into the other's Environment and alerts merge silently, and `.github/tests` cannot read repository settings.

The fix is `external_labels` under Prometheus's `global:`, its value rendered at deploy time, and a regenerated `platform.config-checksum`. Settle: stack (`main-staging`) or environment (`staging`), which are different axes; whether to derive it from `PLATFORM_DEPLOY_HOST` rather than add a secret; and whether Alertmanager routes group on it.

**Kind**: `change`. It needs the stack-or-environment decision and owes an `iac-platform-deploy-pipeline` delta for the new rendered value.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `deploy-the-platform-stack-per-environment` (2026-09-13), which put the stack on a second host; parameterising the Compose file was its declared Non-Goal.

**Re-checked**: 2026-10-19.

## `quieten-the-certificate-expiry-guard-on-a-host-serving-nothing`

`CertificateExpiryNotObserved` (`absent(traefik_tls_certs_not_after{cn!=""}) or count(...{cn=""}) > 0`, from `alert-on-certificate-expiry`) fires on any host that holds no certificate because nothing is routed yet, and reaches Slack. Staging did so from 2026-09-13 until `orders-ops` reached it on 2026-09-16, which cleared that instance without any rule change. Every new host built from `docs/bootstrap-a-new-host.md` meets it between stage 7 and stage 8, so its first alert is noise.

Must not: suppress it wherever no certificate exists, which also silences the case it exists for, a Traefik that served certificates and now serves none. A fix needs a series saying the host is *meant* to serve one (a router count or a per-stack expectation).

**Kind**: `change`. It needs that expectation's design and owes a delta to the certificate-expiry alerting requirement in `iac-platform-services`.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: the operator and a session on 2026-09-15, from the staging host; not by a change.

**Re-checked**: 2026-10-19.

## `aggregate-container-logs`

Logs are read by `docker logs` over SSH, per container, and are lost when a container is recreated, which every deploy does; alerts say *that* a container restarted while the reason is in the log that went away. Low priority for a host running one application, and the first thing missed when it runs several.

Loki with an Alloy (or Promtail) collector reading the Docker socket is the stack-native answer: it joins `platform_monitoring`, Grafana's datasource provisioning pattern applies, retention is bounded as Prometheus's is, and it stores on the data volume under a `platform_data_volume_subdirs` entry. Its prerequisite is delivered: the `docker` role bounds json-file logs at the daemon (`bound-host-log-growth-and-add-swap`), and the collector reads the same files. Traefik's access log is on (`--accesslog=true`, since 2026-09-13), which now shortens Traefik's own `docker logs` history.

**Kind**: `change`. It adds platform services, owing an `iac-platform-services` delta, and needs the collector and retention decisions.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `unknown`.

**Re-checked**: 2026-10-19.

## `check-public-endpoints-from-outside`

Nothing checks from outside a host that a public hostname resolves and answers on 443. Two failures stay unseen. The first is a cloud-firewall change blocking 443 made at the console: the drift workflow reports only what Terraform manages, and UFW is not in 443's path (see `say-what-the-host-firewall-actually-gates`). The second is a DNS failure with no edit behind it, since both zones are served by third-party nameservers (`docs/bootstrap-a-new-host.md` §4.4). An expiring certificate is already covered by the `TLSCertificateExpiringSoon` rule in `platform/docker-compose.yml`. A hostname served no certificate at all produces no `traefik_tls_certs_not_after` series, so that case is not covered.

A host-local `blackbox-exporter` cannot serve this purpose: a probe to the host's own public address is delivered locally and never crosses the Hetzner cloud firewall. What is left is an external uptime service with a check per hostname, independent of the host as the dead-man's switch is. healthchecks.io only receives inbound pings, so this is probably a second vendor account.

**Kind**: `change`. It needs the operator's decision on an external uptime vendor and its cost.

**Waits on**: the operator's choice of an external uptime service.

**Design with**: `nothing`.

**Recorded by**: `unknown`. It was recorded because the monitoring stack watches the host and not the customer's path to it, and it was narrowed by `alert-on-certificate-expiry`, which delivered the certificate-expiry half.

**Re-checked**: 2026-10-19.

## `reconcile-the-heartbeat-checks-with-what-the-repository-prescribes`

What remains is the slug rule, stated where an operator meets it (`docs/bootstrap-a-new-host.md` §7.1 and Appendix A). The periodic-job checks (`infrastructure-drift`, `infrastructure-pre-commit-autoupdate`, `<inventory_hostname>-prune-host-images`) are pinged as `hc-ping.com/<ping key>/<slug>?create=1` by `drift.yml`, `pre-commit-autoupdate.yml` and the `image_prune` role. Re-slugging one at the observer therefore fails silently: the next ping creates a second check under the old slug, and the renamed one goes quiet a period later. Their display names can be changed freely. The per-stack Alertmanager checks are reached only by their own ping URL (`PLATFORM_DEADMANSWITCH_URL`), so renaming one costs nothing. Appendix A's *Renaming a server afterwards* covers the server-rename case, but §7.1 does not tell a reader which checks are free to rename.

It must not undo the following: never re-slug a periodic-job check to tidy the list. The prune slugs are templated from `inventory_hostname`, so changing one at the observer alone puts it permanently out of step with what the host pings.

The rest of the original entry is done. §7.1 now prescribes `<stack>-alertmanager` checks. Appendix A records their period of 5 minutes and grace of 2 minutes, with the reason (an alarm after about seven minutes of silence).

**Kind**: `fix`. It is documentation only, with no decision outstanding.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `correct-the-documents-against-the-tree`. Its operator step looked for a check that the observer did not hold under the prescribed name, and the observer's settings lie outside that change's documents.

**Re-checked**: 2026-10-19.

## `verify-at-deploy-time-that-what-shipped-is-what-runs`

Nothing establishes that a deploy reporting success applied everything it shipped. The proposed check runs after `docker compose up -d --wait`: compare each running container's `com.docker.compose.config-hash` with `docker compose config --hash='*'` and fail on a mismatch. That catches every container that failed to be replaced, whatever the reason. It is a backstop and does not replace the `platform.config-checksum` label: without the label, a configuration-only change hashes equal on both sides.

The check must run on the host, where `.env` is. Several platform services interpolate `${...}` from that file, so a hash computed anywhere else mismatches as an artefact of where it ran. The check edits `app-deploy` (`ansible/roles/deploy_user`), which is generic across applications. It also needs a decision on what a mismatch does to a deploy that has already replaced some services and reported them healthy.

It does not catch a secret rotated inside an embedded config, because both sides hash the same unchanged value. That case is `make-a-rotated-secret-reach-its-inline-config`. A digest taken on the host after interpolation would settle both cases. `apply-shipped-config-on-deploy` considered that deploy-time computation and rejected it in its Decision 1, so the two entries are one decision about the mechanism.

**Kind**: `change`. It changes `app-deploy` for every application, needs the mismatch-handling decision, and owes an `iac-platform-deploy-pipeline` delta to *A Shipped Configuration Change Is Visible to the Container Runtime*.

**Waits on**: `nothing`.

**Design with**: `make-a-rotated-secret-reach-its-inline-config`.

**Recorded by**: `apply-shipped-config-on-deploy`, which deliberately stops at making a change visible. Checking that the change was applied edits the generic `app-deploy` and needs a decision of its own.

**Re-checked**: 2026-10-19.

## `read-the-rendered-env-back-through-the-host-s-own-parser`

`platform-deploy.yml` escapes each value it writes into `platform/.env` (double quotes, then `\` → `\\`, `"` → `\"`, `$` → `$$`, in that order) so that Compose's dotenv parser reconstructs it. That rule belongs to one version of the parser: it was measured on Compose v5.4.0 while the host ran v5.5.0, and both versions move under Dependabot and host package updates. A parser that diverges does not fail the deploy. The service starts, reports healthy, and fails wherever it needed the credential. `.github/tests/test_the_rendered_env_survives_its_parser.py` checks only that the escaping exists and runs in order.

The fix is a round trip at deploy time, on the host: render the file, parse it back with the host's own Compose, compare digests rather than values so that nothing is disclosed, and refuse the deploy on any mismatch. `tools/env-rendering-probe/` already holds the harness, both consumption paths and the value corpus.

**Kind**: `change`. It adds a deploy-time refusal, which owes an `iac-platform-deploy-pipeline` delta to *Platform Secrets Rendered from CI at Deploy Time*.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `render-the-env-file-so-a-secret-survives-it`. That change measured the rule but could not arrange for a divergence to be noticed, and the check belongs to the deploy path.

**Re-checked**: 2026-10-19.

## `escape-a-value-for-the-configuration-it-is-interpolated-into`

The `.env` escaping guarantees only what the dotenv parser yields. `SLACK_WEBHOOK_URL` and `DEADMANSWITCH_URL` are then interpolated unquoted into the embedded Alertmanager YAML in `platform/docker-compose.yml`, and `ACME_EMAIL` goes into a Traefik `command:` argument. A `: `, a `#`, a quote or a line break in one of these values would change or break the document it lands in, and nothing escapes for either context. Today's vendor-issued values carry no such character, but that is a property of the values and not of the file, so this work is still owed. `platform/README.md` states the residual.

One decision settles it: escape at render time, with a second escaper per destination, or move the two URLs out of the embedded configuration through Alertmanager's file reference, which removes the question. Either way, a corrected value reaches the container only when the container is next replaced (*A Shipped Configuration Change Is Visible to the Container Runtime*, `openspec/specs/iac-platform-deploy-pipeline/spec.md`).

**Kind**: `change`. It needs the escape-or-file-reference decision and owes an `iac-platform-deploy-pipeline` delta.

**Waits on**: `nothing`.

**Design with**: `make-a-rotated-secret-reach-its-inline-config`, because moving the URLs out of the embedded config changes how a rotation reaches the container.

**Recorded by**: `render-the-env-file-so-a-secret-survives-it`, whose obligation deliberately stops at what the dotenv parser yields.

**Re-checked**: 2026-10-19.

## `assert-no-secret-is-built-into-a-url`

postgres-exporter once built `DATA_SOURCE_NAME` by interpolating a password into a URL, and a `#` in that password discarded the host (see `alert-on-the-exporter-being-unable-to-read-postgres`). The service now uses `DATA_SOURCE_URI`, `DATA_SOURCE_USER` and `DATA_SOURCE_PASS`. What is still owed is a static check in `.github/tests` that stops the shape from returning.

The rule is: a secret-bearing variable inside a URL, within its own whitespace-delimited word.

- Offences: a scheme URL; a `user:secret@host` authority with no scheme; a query parameter; a path segment of an authority with no scheme; text concatenated onto a secret that is itself a URL.
- Not offences: a value that is only the secret (Alertmanager's `api_url`); a non-secret host; a secret next to an unrelated URL; a filesystem path; Compose's `$$`.

An earlier draft repeated one defect across three review rounds. Each arm must read the quote-stripped text around the secret rather than anchor to the start of the raw word, because quoting inside `configs:` and `--flag=value` inside `command:` are both idiomatic.

Secret names must be read from the `env:` block of `platform-deploy.yml`'s render step, not guessed from keywords: `DEADMANSWITCH_URL` is a bearer credential although nothing in its name says so. The scan must cover `command:`, `entrypoint:`, `labels:`, `healthcheck.test`, `environment:` and the inline `configs:`, since `${ACME_EMAIL}` appears only in `command:`. The check must not be able to pass having read nothing, so it asserts a floor on the number of interpolations it found.

**Kind**: `change`. It owes a requirement stating the rule the check enforces (`iac-platform-deploy-pipeline`), and its earlier draft did not fit one review sitting.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `unknown`. It was split on 2026-09-15 from the fix to postgres-exporter's connection settings, because the check's draft was still wrong after three review rounds.

**Re-checked**: 2026-10-19.

## `hold-every-platform-deploy-for-the-length-of-a-window`

`platform/README.md`'s *Upgrading the PostgreSQL major version* asks the operator to hold every other `platform/**` merge and every `platform-deploy.yml` dispatch for the length of the window. Nothing enforces that, and the failure is silent. Step 2 leaves the host without the `postgres` container its Compose definition declares. Any platform deploy that lands in the gap, whether an unrelated merge or the rebuild dispatch in `docs/runbook-rebuild.md`, recreates the container on the old major with an empty cluster, which undoes step 2 without an error. The window's own pull request is already gated, because it is opened as a draft and marked ready only after the store is cleared. A draft gates only itself.

The class is wider than PostgreSQL. The platform stack is deployed wholesale on every merge, so a deploy can undo any window that leaves the host deliberately out of step with the committed definition. `platform-deploy.yml`'s per-stack `concurrency` group serialises deploys but does not refuse one. A refusal needs state that outlives a run: a repository variable, an issue label, a file on the host, or an Environment gate. The host already carries an operator-declared window file, `/var/lib/platform-maintenance/shared-postgres-window`, which `platform-deploy.yml` does not read today. The design must cover a window abandoned half-way and decide what a held merge does: queue and land unwatched at release, or fail loudly and be re-run.

**Kind**: `change`. It needs operator decisions on where the lock lives, who releases it, and whether a held deploy queues or fails.

**Waits on**: `nothing`.

**Design with**: `notice-a-pending-reboot-and-decide-what-does-it`. An automatic-reboot policy is a scheduled outage, and this mechanism is what would keep a deploy out of it.

**Recorded by**: `unknown`. It was recorded on 2026-09-15 from the work that gated the window's own pull request with a draft. That half was a fix, while this half needs a mechanism and decisions.

**Re-checked**: 2026-10-19.

## `automate-per-application-database-provisioning`

*Single Shared PostgreSQL Instance, Per-Application Databases* (`openspec/specs/iac-platform-services/spec.md`) obliges this repository to automate two things: how an application's database and role are provisioned in the shared instance, and how the role's password reaches the application. The trigger fired on 2026-09-13, when `orders-ops` got a database on staging, and production followed. Both were provisioned by hand from the recipe in `docs/onboard-an-application.md`. The requirement carries a paragraph, "One divergence is stated rather than hidden, as of 2026-09-13", saying the obligation stays unmet until this mechanism lands.

A design must answer:

- Where is the password generated, and how does it reach the application's deploy without a command line, a log line or a file in this repository?
- What happens when the secret name is already taken? Production's `orders-ops` Environment already held `POSTGRES_PASSWORD`, so the shared-instance password became `SHARED_POSTGRES_PASSWORD`, and the mechanism must handle that collision for every application.
- A re-run is a rotation, which the application picks up only on its next deploy.
- How is a database re-provisioned after a rebuild or a store reset?
- Each host's credential must be independent, so that one leaked credential reaches one host.

**Kind**: `change`. It owes a MODIFIED delta on *Single Shared PostgreSQL Instance, Per-Application Databases* that replaces the divergence paragraph, an update to the recipe in `docs/onboard-an-application.md`, and a design of the credential path.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `provision-orders-ops-database-in-the-shared-instance` (archived 2026-09-15). It provisioned by hand because a mechanism fitted to one consumer would be found wrong by the next.

**Re-checked**: 2026-10-19.

## `classify-orders-ops-production-data-for-the-shared-instance`

On 2026-09-14 the operator decided that production's `orders-ops` data leaves its private PostgreSQL for the database already provisioned in production's shared instance, and moves to ExamplePG only later. That data includes hand-curated rows, and the operator classified it as tolerable to lose, to the operator, with no backup. The policy is temporary, taken because the application is at an early, experimental stage, and it ends with the move to ExamplePG. *Single Shared PostgreSQL Instance, Per-Application Databases* (`openspec/specs/iac-platform-services/spec.md`) still keeps durable data out of the shared instance unconditionally, so the specification does not yet record this decision.

The change owes a MODIFIED delta on that requirement, shaped like its staging rehearsal-data policy: whose loss is tolerable, which application and host the policy covers, and that it ends with the move to ExamplePG. It also owes a delta on *No Store on This Host Holds Data Requiring Backup* (`openspec/specs/iac-safety-hardening/spec.md`). It must merge before any production row lands in the shared instance, and the `orders-ops` repository's production cutover waits on it. `platform/README.md`'s major-upgrade section already warns that once the data moves in, clearing production's store deletes it for good. `docs/runbook-rebuild.md` does not carry that warning yet, and its procedure needs a re-read once this change lands.

It must not undo the following: the classification names one application on one host and gives no permission for any other application's durable data.

**Kind**: `change`. It owes MODIFIED deltas on *Single Shared PostgreSQL Instance, Per-Application Databases* and *No Store on This Host Holds Data Requiring Backup*.

**Waits on**: `nothing`.

**Design with**: `move-orders-ops-durable-data-to-examplepg`, whose step 2 is this change.

**Recorded by**: the operator's decision of 2026-09-14, taken after `provision-orders-ops-database-in-the-shared-instance` had merged, so that change could not carry it.

**Re-checked**: 2026-10-19.

## `move-orders-ops-durable-data-to-examplepg`

*No Store on This Host Holds Data Requiring Backup* (`openspec/specs/iac-safety-hardening/spec.md`) states one divergence, as of 2026-09-08: on production, `orders-ops` keeps durable data in a PostgreSQL container of its own on its `app_db` network, and nothing backs it up. `docs/runbook-rebuild.md` repeats this. Nothing else tracks the divergence. The operator decided on 2026-09-14 to resolve it in two stages. First, the whole database moves into production's shared instance. That includes the `procrastinate_*` queue tables, because the application enqueues inside its domain transaction. Later, the database moves to ExamplePG, which owns its own backups.

Ending the private container takes four steps:

1. Done by `provision-orders-ops-database-in-the-shared-instance`: the role, the database and `SHARED_POSTGRES_PASSWORD` (deliberately not `POSTGRES_PASSWORD`).
2. `classify-orders-ops-production-data-for-the-shared-instance`, in this repository.
3. The `orders-ops` repository's own migration. It is a dump-and-restore, and `DATABASE_URL` is repointed to `orders-ops@postgres:5432/orders-ops`. The application's PostgreSQL service, its volume and its `app_db` network are removed in the same deploy, because while both exist `postgres` names two servers.
4. Here: delete the divergence paragraph and the runbook's matching paragraph.

Gate step 4 on the host, not on a report: on production, `docker ps --filter name=orders-ops` shows no PostgreSQL container, and `docker volume ls` no longer lists `orders-ops_orders_ops_pgdata`.

For the later move to ExamplePG, use the session pooler `db-pooler.example.net:5432` with user `postgres.<ref>`, which answered `pg_isready` from staging's `platform_edge` on 2026-09-14. The pooler is needed for three reasons: the direct connection is IPv6-only while `platform_edge` has IPv6 disabled, the transaction pooler on port 6543 cannot carry the worker's `LISTEN/NOTIFY`, and the Free plan has no backups.

**Kind**: `change`. It owes a MODIFIED delta on *No Store on This Host Holds Data Requiring Backup* that removes the divergence. The migration itself is the `orders-ops` repository's work.

**Waits on**: `classify-orders-ops-production-data-for-the-shared-instance` merging, and then the host check above showing that the `orders-ops` repository has removed its private PostgreSQL.

**Design with**: `classify-orders-ops-production-data-for-the-shared-instance`.

**Recorded by**: `unknown`. It was recorded because the specification names the divergence and nothing else tracked it, and the migration belongs to the `orders-ops` repository, outside this one's authority.

**Re-checked**: 2026-10-19.

## `exercise-the-volume-server-coupling-against-live-state`

`terraform/stacks/main-production/main.tf` couples the volume to the server with `count = var.volume_enabled && var.server_enabled ? 1 : 0`. *Conditional Prod Volume Creation* (`openspec/specs/iac-data-volumes/spec.md`) states two scenarios against it: *Volume toggle disabled creates nothing* and *Disabling the server also removes the volume*. Nothing has ever run either against production's live state, and the module tests under `terraform/modules/volume/tests/` cannot reach a stack's `count`.

Staging's half was observed on 2026-09-16 by `write-and-rehearse-the-rebuild-runbook`. With `server_enabled = false`, its pull request's plan showed `0 to add, 0 to change, 3 to destroy` (server, firewall and volume), and the apply destroyed the volume first. That rehearsal also settled where the reads can run: the pull request's own plan job. A local plan cannot run, because every stack's `cloud { organization = "acme" }` needs an HCP credential the workstation lacks. What remains is production, where no backup covers the volume. It takes two plan-only reads, each setting one toggle to false, on a pull request that is read and then closed. That pull request is **never merged or applied**.

**Kind**: `fix`. It owes no delta and needs no decision: it runs two existing scenarios as plans and records the result.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `make-openspec-validation-a-usable-gate`, which recovered it from `add-prod-data-volume`'s unticked task 3.5. That task sat inside an archived record, where it would otherwise have been lost.

**Re-checked**: 2026-10-19.

## `say-what-a-stale-saved-plan-is-and-how-to-recover-from-it`

`apply.yml` names its failures and the recovery for most cases, but a stale saved plan only gets Terraform's raw `Error: Saved plan is stale`. The refusal itself is the gate working: *Gated Production Apply Applies the Reviewed Plan* demands the exact plan a human reviewed. What the change owes is a named failure for this case. It should say what staleness means, that no infrastructure change was lost, and that recovery is a **fresh run of the whole workflow**, because re-running the failed job re-downloads the same stale artifact. It must not re-plan automatically, which would apply a plan nobody reviewed.

The case is likely because production's apply waits for a required reviewer, and that wait is the window in which the plan goes stale. It was observed on 2026-09-12, on the merge of `rename-the-external-services` (PR #155). Production's state serial moved from 44 to 45 between plan and apply, with no attribute changed, which shows that even a no-op operation bumps the serial. A local `terraform plan` under the read-only token was measured not to bump it. CI, the operator's checkout, the change's working trees and HCP health assessments were all ruled out as the cause. The one avenue not yet read is the workspace's **States** view in HCP, which labels each state version with its source.

**Kind**: `change`. It needs a decision on whether the message names likely causes. If a permitted local operation turns out to invalidate a pending apply, it also needs an `AGENTS.md` rule change.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: the merge of `rename-the-external-services` (PR #155), whose production apply failed this way. The error-handling gap lay outside that change's subject.

**Re-checked**: 2026-10-19.

## `make-a-waiting-approval-announce-itself`

Nothing tells the operator that a gated run is waiting for approval. On 2026-09-12 a production converge sat unapproved from 06:26 to 09:56, and on 2026-09-13 another waited 6h34m. Each was found only because a later run queued behind it with `waiting on converge (main-production) … to complete`, which reads like a hung job. The Terraform apply, the host converge and the platform deploy gate on the same Environment and render prompts that cannot be told apart. A pending approval also stalls every later converge of that stack.

Before building anything, measure whether GitHub's *deployment review requested* notification exists and simply does not reach the operator. If it does, the fix is a settings change plus a sentence in `docs/bootstrap-a-new-host.md` §6.6. Otherwise, build a scheduled query of `gh run list --status waiting` that pushes to somewhere the operator reads, and repeats while the state persists. There is a trap: Alertmanager's `SLACK_WEBHOOK_URL` comes from `PLATFORM_SLACK_WEBHOOK_URL`, a production-Environment secret. A notifier using that secret would itself wait on the approval it is meant to announce, so its credential must come from somewhere ungated. It must not fire on every run.

**Kind**: `change`. It needs operator decisions on the channel and the credential source, and may add a scheduled workflow, which carries a heartbeat obligation.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `rename-the-external-services`, which lost three and a half hours to the problem and recorded only advice on recognising it.

**Re-checked**: 2026-10-19.

## `factor-the-four-stack-discovery-bodies`

`pr-validation.yml`, `apply.yml` and `drift.yml` share one stack-discovery shell body of about 160 lines. `.github/tests` asserts that the three copies are identical: it locates each body by the `terraform/stacks` path literal it contains and executes it against a scratch tree. `host-converge.yml` and `platform-deploy.yml` now carry similar bodies that are deliberately not identical, and both sit outside the identity assertion. `host-converge.yml` enumerates `ansible/inventory/*.hcloud.yml` and cross-checks it against the stacks, and `platform-deploy.yml`'s comment counts "Five near-siblings".

A script under `.github/scripts/` (none exists yet) is likely simpler than a composite action, because the tests would execute it directly instead of locating it by a path literal. The shared parts are the flat `key: value` reader, the secret-name validation, the empty-result refusal and the JSON emission. The roots and the cross-checks stay parameters. The cost to weigh is that the diff restructures the production apply path and must replace the identity assertion rather than retarget it.

**Kind**: `change`. It restructures every gated workflow's discovery and replaces a test assertion, which is too large for one review sitting.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `apply-host-configuration-through-a-gated-workflow`, which wrote the fourth body (`host-converge.yml`). Factoring was out of scope for a change that was already restructuring gated workflows.

**Re-checked**: 2026-10-19.

## `two-deferred-ci-items`

`close-ci-verification-gaps` deferred two unrelated CI improvements:

- **The destroy-policy gate's inspection logic is inline workflow shell.** Moving it into a version-controlled script with executable fixtures would make the highest-consequence logic in this repository testable as code. That change's `design.md` Decision 5 deferred it on scope, not on merit. Its `tasks.md` 1.1 describes four fixtures: clean, destructive, malformed, and valid JSON that is not a plan. `.github/tests/test_ci_configuration.py` asserts that the routes are closed, not that each one is reached.
- **`actionlint` is cited as a verification means, but nothing installs it.** It is not in `.pre-commit-config.yaml`. Adding the hook first needs a decision on two existing, intentional `SC2016:info` findings: single-quoted markdown in an `echo`, in `pr-validation.yml`'s plan-comment step and in `apply.yml`'s job-summary step.

A third item was taken as a fix on 2026-09-15: `pre-commit-autoupdate.yml` now installs the pinned `pre-commit` from `.github/requirements-ci.txt`.

**Kind**: `change`. The first item restructures the production destroy gate, and the second needs an operator decision on how to handle the two findings.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `close-ci-verification-gaps`, which deferred both on scope rather than rejecting them.

**Re-checked**: 2026-10-19.

## `matrix-the-molecule-suite-over-scenarios-and-bound-each-job`

`molecule test --all` stops at a role's first failing scenario and leaves the remaining scenarios out of the SCENARIO RECAP, so a red run under-reports while looking complete. Molecule's own remedy, `--continue-on-failure`, needs `--workers`, and `--workers` refuses outside collection mode (observed 2026-09-07). The remedy that works is an `ansible-verify.yml` matrix over **scenarios** rather than roles; today the job is `molecule (${{ matrix.role }})`. It also parallelises the longest role. This is not the shared-state hazard, which per-working-tree namespacing has already closed.

The same change bounds each job. No workflow sets `timeout-minutes`, so every job inherits GitHub's 360-minute default. Because `ansible-verify` is a required check, a hung scenario leaves a pull request pending for six hours, and `AGENTS.md` describes ways a Molecule run hangs. The timeout must sit above the slowest run that still passes and below six hours. Choose it against durations measured after `cache-the-apt-index-within-a-converge`, and record them in the proposal.

It must not undo two requirements in `openspec/specs/iac-cicd-pipeline/spec.md`:

- *Required Status Checks Report on Every Pull Request*: the aggregating job with a literal name stays as the registered check.
- *Ansible Configuration Is Verified in Continuous Integration and Gates the Merge*: scenarios are discovered from `ansible/roles/*/molecule/*/` without a workflow edit, and an empty result fails loudly.

**Kind**: `change`. It reshapes the jobs behind a required check and needs an operator decision on the timeout value.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `promote-molecule-to-a-required-check`, which named it a non-goal so that the change registering the check did not also redefine it. The timeout half was recorded on 2026-09-11, alongside `cache-the-apt-index-within-a-converge`.

**Re-checked**: 2026-10-19.

## `test-a-play-at-play-scope`

`ansible/playbooks/host-baseline.yml` has a guard play that refuses when the target environment resolves to no host. Nothing tests it:

- Molecule's subject is a role on a host.
- `.github/tests` reads only committed files, and `TestEveryModuleInTheSuiteDirectoryNeedsNoPrivilegedResource` would fail a module that ran `ansible-playbook`.

So a green pull request shows only that the guard is present. One bypass is known and cannot be closed from the play: `--limit` filters `localhost` out of the guard play, so `--limit <host>` against an empty group exits 0. The play's header and `docs/bootstrap-a-new-host.md` state this bypass.

A play-scope harness would assert the refusal under each way of invoking the playbook. It would also cover other play-level behaviour: role ordering, `when:` conditions on role inclusion, and the deferred play-scope input validation noted above the playbook's role list.

**Kind**: `change`. It adds a fourth test command (a row in `AGENTS.md`'s test-command table) and a runner for it, which is an operator decision.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `configure-the-staging-host` (`design.md` Decision 10). That change's diff needed close reading and was no place to invent a test tier.

**Re-checked**: 2026-10-19.

## `tighten-the-refusal-scenario-s-own-filesystem-assert`

`ansible/roles/platform_data_volume/molecule/superseded-path-in-force-refused/verify.yml` asserts `platform_data_volume_blkid.rc != 0` to show that the role refused before formatting the device. This assertion fails open. `blkid` also returns 1 on a usage error, so a probe that never happened reads as a refusal. Only rc 2 means "looked and found nothing", and the same role's `prepare.yml` already uses `rc != 2`.

A second hazard in the same assertion remains after that fix. `prepare.yml` and `verify.yml` each redeclare `platform_data_volume_fixture_loop_minor` (`base + 4`) by hand. If `verify` drifts, it probes an unassociated minor, gets rc 2 and passes. Either derive the minor from one place, or have `verify` assert that its device is the one `prepare` associated. Fix the rc-1 case, and do not treat the drift as closed by it.

**Kind**: `fix`. It is a test-only tightening that owes no delta and needs no decision.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `namespace-the-molecule-loop-devices`. Its code review found the problem in a line that change did not author, and it was left out of the diff to avoid another review round.

**Re-checked**: 2026-10-19.

## `adopt-the-stubbed-runtime-rig-for-the-guards-molecule-cannot-reach`

Two guards in the `image_prune` role have no assertion covering them:

- Local images are enumerated before the keep set is computed.
- Each tag is re-resolved immediately before removal.

Both are observable only when the host's images change during a run, and a black-box Molecule scenario has no point at which to change them. These two guards close the concurrent-deploy window against `app-deploy`, the only actor competing with the prune. The two abandon branches added by that change's review fix are also unasserted.

That change's code review built a rig: a stubbed `docker` on `PATH` that answers some calls and fails others. The rig showed both guards present and mutation-visible. That change's `test-plan.md` invites adding a deterministic arrangement ("sized rather than slept") to `tasks.md` 2.7 and 2.8. Nothing under `ansible/roles/image_prune/molecule/` uses such a stub yet.

**Kind**: `fix`. It adds tests for behaviour that is already specified, with no delta and no decision.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `prune-unreferenced-host-images-periodically`. Its code review built the rig to verify the change, not to ship it.

**Re-checked**: 2026-10-19.

## `hold-the-whole-static-suite-to-its-own-constraints`

The self-check half is done: `TestEveryModuleInTheSuiteDirectoryNeedsNoPrivilegedResource` in `.github/tests/test_ci_configuration.py` now holds every module in `.github/tests` to the no-network, pinned-imports-only, no-container and no-Terraform constraints, beside the older `TestTheSuiteNeedsNoPrivilegedResource`, which still reads only its own file (`SUITE_PATH`). What remains is the half absorbed from `prune-the-repository-walkers-of-provisioned-content`: repository-scope walkers read a directory rather than the committed tree.

`compose_shaped_files()` (behind `TestEveryComposeFileDeclaringAServiceImageIsCovered`) walks through `walked_files()`, whose prune lists (`PRUNED_ANYWHERE`, `PRUNED_AT_ROOT_RELATIVE`) do not include `.molecule-home/`. On a working tree provisioned for Molecule it finds the `community.docker`/`community.general` fixture Compose files there and fails; CI never provisions Molecule and stays green. It is also a race while Molecule creates and removes `tmp/` under that directory (observed 2026-09-11). The fix is to point repository-scope walkers at the existing `tracked_files()` helper in the same module, which closes the class (also `.terraform/`, virtual environments, sibling working trees), not to prune one more directory. **Must not undo**: deleting `.molecule-home/` to get green throws away the provisioning the Molecule tier depends on.

**Kind**: `fix`. No delta and no decision: the helper and its admitted `git` use already exist.

**Waits on**: `nothing`.

**Design with**: `share-the-static-suites-test-arrangement`, which gives the same suite a support module.

**Recorded by**: `alert-on-certificate-expiry` (the self-check half, as the change that added a second suite module), and the test author deriving `cache-the-apt-index-within-a-converge` (the walker half, met as a red suite); neither change's subject was the suite's walkers.

**Re-checked**: 2026-10-19.

## `complete-the-autoupdate-workflow-s-own-checks`

`open-autoupdate-pr-with-app-token` added a section to `.github/tests/test_ci_configuration.py` that finds pull-request-opening workflow steps and asserts the identity they use. Its helpers have four reading limits, each producing a false positive on a legitimate change: `token_inputs` reads only step-level `env:` and misses a job- or workflow-level `GH_TOKEN` (the likeliest to bite, the first time a workflow opens a pull request with `gh`); `step_opens_a_pull_request` matches a POST and `/pulls` anywhere in one `run:` block rather than one command; `secrets_referenced_by` scans the whole job rather than the identity path; `readme_sections` splits at any `#` line, including inside a code fence.

The section also asserts nothing about two properties of `.github/workflows/pre-commit-autoupdate.yml`: the minting step's `permission-contents: write` / `permission-pull-requests: write` down-scoping, and the restricted input set on the pull-request step (no `labels`/`assignees`, which need Issues). Neither is live while the App's own scope is exactly those two permissions. Both come from a prose clause, not a scenario, so a scenario must be written before a test can be derived. The workflow header states which of its claims the suite does not stand behind; that note is what this change replaces.

**Kind**: `change`. It owes a delta adding scenarios for the two unasserted properties, and it edits tests an independent author derived from delta specs.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `open-autoupdate-pr-with-app-token`, whose tests were independently derived from its deltas; correcting or extending them was a change of its own. Merged here on 2026-09-16 from `widen-what-the-pull-request-identity-checks-can-read` and `assert-the-autoupdate-workflow-s-two-unchecked-properties`.

**Re-checked**: 2026-10-19.

## `assert-every-role-has-a-mock_roles-entry`

`.ansible-lint`'s `mock_roles:` hand-lists every role referenced by name, because ansible-lint does not resolve role references through `ansible/ansible.cfg`'s `roles_path`. Nothing asserts the list is complete. A role added under `ansible/roles/` and referenced from `ansible/playbooks/host-baseline.yml` without an entry fails ansible-lint twice with a false-positive "role not found" naming a search path, which reads as a configuration problem rather than a missing line (observed 2026-09-08 adding `swap`). The list is complete today.

The check is a static read for `.github/tests`: the directories under `ansible/roles/` minus the gitignored external role, against `mock_roles`. It has the same shape as the existing "every lockfile-bearing directory is covered" check.

**Kind**: `fix`. A static assertion with no delta or decision.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `bound-host-log-growth-and-add-swap`, which added its own missing entry; asserting the invariant was a different concern and the suite was not that change's subject.

**Re-checked**: 2026-10-19.

## `assert-every-tfvars-assigns-its-required-variables`

Nothing checks that each stack's `terraform.tfvars` assigns every variable its `variables.tf` declares without a `default`. A gap surfaces only at plan time: `pr-validation.yml`, `apply.yml` and `drift.yml` run Terraform with `-input=false`, so the plan fails `No value for required variable`, the required check fails, and the nightly drift sweep fails for that stack every night and takes the shared heartbeat with it. That detection needs a credential, a workspace and a network; the same fact is a static read that belongs in `.github/tests`. The only real work is a small parser for `variable` blocks and `terraform.tfvars` assignments without HCL machinery the suite lacks; `test_a_second_environment.py` already reads `terraform.tfvars`. Both stacks assign every required variable today.

**Kind**: `fix`. A static assertion with no delta or decision.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `add-a-staging-environment`'s code review (2026-09-10), which met the gap as that change's own in-flight state (`server_type` unassigned in `main-staging`); a check failing the tree being assembled belonged to a later change.

**Re-checked**: 2026-10-19.

## `assert-markdown-prose-is-not-hard-wrapped`

`AGENTS.md` says "Do not hard-wrap prose. Keep each paragraph on a single line whatever its length", and nothing checks it. Hard wraps re-accumulate silently: the sweep beside which this was recorded rewrapped 72 of 211 tracked markdown files. Under the rule, any prose paragraph spanning more than one line is a violation, with no threshold to tune. The detector must exempt the breaks Markdown requires: fenced and indented code, tables, YAML frontmatter, list-item continuations, and a line ending in two spaces or a backslash.

Scope is markdown prose only; comments in `.py`, `.yml`, `.sh` and `.tf` files stay wrapped. It is not a formatter: Prettier's `proseWrap: "never"` would rewrite style across the tree and archived records, and there is no Node toolchain for it; markdownlint has no rule for the opposite of `MD013`. It fits `.github/tests` as one new module.

**Kind**: `fix`. A static assertion with no delta or decision.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: unknown change; recorded 2026-09-11 beside the fix that removed the hard wraps, which wrote the rule but not its check.

**Re-checked**: 2026-10-19.

## `unify-the-workflows-own-role-discovery`

`.github/workflows/ansible-verify.yml`'s *Discover roles carrying scenarios* step enumerates roles in shell by the directory-name heuristic (`grep -v '\.'`). `unify-the-two-role-exclusion-rules` replaced that heuristic in both Python enumerations (`.github/tests/test_ci_configuration.py` and `ansible/scripts/select_molecule_roles.py`) with the rule derived from `ansible/requirements.yml`. The three agree on today's tree; they diverge on a second pin in `src:` form or on vendored content nobody pinned. No assertion compares the workflow's enumeration with the others.

The smaller option is taking the role list from the selector. Expect `test_the_matrix_runs_the_roles_a_pull_request_owes.py`'s `_discovery_steps` fixtures, which build scratch trees with no `ansible/requirements.yml`, to meet the selector's `DerivationRefused`. The same module's `test_an_external_galaxy_dependency_contributes_no_edge_and_is_not_refused` still filters with `if "." in target`; it must be rewritten with a new way to express "an edge to content this repository does not carry", which is the same question this entry asks of the workflow.

**Kind**: `change`. It changes what CI runs and needs a decision on how the workflow obtains the manifest-derived list.

**Waits on**: `nothing`.

**Design with**: `bind-the-galaxy-name-resolution-to-ansible-itself`, which works on the same resolution of external role names from the same manifest.

**Recorded by**: `unify-the-two-role-exclusion-rules`; nothing bound the workflow to the other two rules, and changing a workflow's discovery was not a rider on a test-helper change.

**Re-checked**: 2026-10-19.

## `bind-the-galaxy-name-resolution-to-ansible-itself`

Two implementations resolve a `roles:` entry to the directory `ansible-galaxy` installs it under, in `.github/tests/test_ci_configuration.py` and `ansible/scripts/select_molecule_roles.py`, and a test binds them only to each other. Both were once wrong in the same two ways with every test green: they refused `- role: <name>`, and they split a `,version` qualifier before stripping `.git`, so `…/ansible-role-docker.git,8.0.0` was named `ansible-role-docker` instead of `ansible-role-docker.git`. Both were corrected by hand against Ansible 2.21.3; nothing checks them against `RoleRequirement.role_yaml_parse`.

`.github/tests` cannot import `ansible-core`: *The Suite Needs No Privileged or External Resource* (`openspec/specs/iac-cicd-pipeline/spec.md`) limits it to the standard library and `.github/requirements-ci.txt` (`pre-commit`, `PyYAML`), and the suite catches even a lazy import. The check belongs where Ansible is pinned: a Molecule scenario, or an `ansible-verify.yml` step after `ansible/requirements-test.txt` is installed. Adding `ansible-core` to `requirements-ci.txt` instead changes what the static suite is.

**Kind**: `change`. It needs a decision on where the check lives; the `requirements-ci.txt` option would owe a delta on *The Suite Needs No Privileged or External Resource*.

**Waits on**: `nothing`.

**Design with**: `unify-the-workflows-own-role-discovery`.

**Recorded by**: `unify-the-two-role-exclusion-rules`, which found both copies wrong but could not write the check inside the static suite.

**Re-checked**: 2026-10-19.

## `catch-up-the-drifted-galaxy-pins`

`ansible/requirements.yml` pins `hetzner.hcloud` 7.0.0 and `geerlingguy.docker` 8.0.0 (current on 2026-09-08). Three collections are at least a major version behind: `community.general` 9.5.0 (13.4.0 then), `ansible.posix` 1.6.2 (2.2.2), `community.docker` 4.1.0 (5.3.0). Dependabot has no `ansible-galaxy` ecosystem, so nothing proposes bumps here. They are used by `hardening` (`community.general.ufw`), `deploy_user` (`ansible.posix.authorized_key`'s `key_options`, `community.docker.docker_login`), `swap` (`ansible.posix.mount`, `ansible.posix.sysctl`) and the dynamic inventory. Molecule is the gate: `molecule test --all` per role, reading the SCENARIO RECAP rather than the exit code. `ansible-core` (pinned in `ansible/requirements-test.txt` and `ansible/requirements.txt`) and these collections were verified together as one set.

**Kind**: `change`. A four-major migration across several collections, too large for one review sitting.

**Waits on**: `nothing`.

**Design with**: `cover-the-unwatched-manifests-and-settle-the-watcher`, since `ansible-core` and the collections are a matched set and a bot bump landing mid-migration would confuse which half broke.

**Recorded by**: an inventory taken while archiving `open-autoupdate-pr-with-app-token` (2026-09-08); a migration was not a rider on that archive.

**Re-checked**: 2026-10-19.

## `cover-the-unwatched-manifests-and-settle-the-watcher`

`.github/dependabot.yml` watches `terraform`, `github-actions`, `docker-compose` and `npm`. Nothing watches the pip manifests: `.github/requirements-ci.txt`, `ansible/requirements-test.txt` and `ansible/requirements.txt`. `ansible/requirements.txt`'s `ansible-core` must equal `requirements-test.txt`'s and `.github/tests` fails on a difference, so a bot opening a pull request against one would be red on arrival: the two must move as a pair, by a grouped update or by watching one and asserting the other follows. Do not inherit `cover-platform-images-with-dependabot`'s "one stanza" estimate: *Automated Dependency Updates* (`openspec/specs/iac-safety-hardening/spec.md`) enumerates ecosystems by name and the suite reads that back, so a new ecosystem is a stanza plus a delta plus its tests.

The watcher question is settled with a recommendation: stay with Dependabot for now. Renovate covers `pre-commit` and `ansible-galaxy` natively, but hosted Renovate is a third party with write access; self-hosted Renovate could reuse the existing autoupdate GitHub App. Revisit when a second Compose stack appears, the galaxy manifest grows past a handful of entries, or the galaxy migration is done and small bumps start being missed. The old precondition, that something notices a red scheduled run, is met by `notice-when-a-periodic-job-stops-reporting`'s heartbeat checks.

**Kind**: `change`. It owes a delta widening *Automated Dependency Updates*' ecosystem enumeration, and a decision on how the two `ansible-core` pins move together.

**Waits on**: `nothing`.

**Design with**: `catch-up-the-drifted-galaxy-pins`.

**Recorded by**: unknown change; it is the same shape as `cover-platform-images-with-dependabot`, which closed the image half, and the watcher question was recorded separately on 2026-09-08.

**Re-checked**: 2026-10-19.

## `rename-the-requirements-that-read-narrower-than-they-are`

Requirement names are this repository's citation form, and several still name production or "this host" alone although two stacks now share them. Left by `make-the-pipeline-environment-agnostic`: *Gated Production Apply Applies the Reviewed Plan* (a genuine name/content mismatch) and *Credential Scoping by Privilege* (`iac-cicd-pipeline`), *Write Credentials Confined to the Gated Pipeline* (`iac-safety-hardening`). A second set: *Conditional Prod Server Creation* (`iac-server-lifecycle`), *Conditional Prod Volume Creation* (`iac-data-volumes`), *Data Durability for Stateful Resources* and *No Store on This Host Holds Data Requiring Backup* (`iac-safety-hardening`), *Host Joins a Private Tailnet for Non-Operator SSH Access* and *Unprivileged Operator Accounts Support Interactive Host Inspection* (`iac-host-configuration`). Staging uses the same toggles, tailnet and operator account, and staging took the platform stack on 2026-09-13 and an `orders-ops` database the next day, so it holds stores: `platform_postgres_data`, `platform_traefik_letsencrypt` and Alertmanager's volume; "this host" is now wrong about which host a durability obligation binds.

Renaming sweeps every citation: comments in `apply.yml`, `drift.yml` and `pr-validation.yml`, `docs/bootstrap-a-new-host.md`, `.github/tests`, and archived records. `.github/tests/test_a_retired_requirement_name_is_reported.py` makes the sweep checkable. Precedent: `add-a-staging-environment` renamed *Each Stack Has a Dedicated Hetzner Cloud Project* where it had a content change to carry it.

**Kind**: `change`. It owes rename deltas across four capabilities and a repository-wide citation sweep.

**Waits on**: `nothing`.

**Design with**: `sweep-the-stale-scenario-titles-and-check-them`, the same citation-sweep mechanism over scenario titles.

**Recorded by**: `make-the-pipeline-environment-agnostic` (first set) and the staging work that followed (second set); both declined to mix a mechanical rename into diffs that most needed reading closely.

**Re-checked**: 2026-10-19.

## `sweep-the-stale-scenario-titles-and-check-them`

Citations here name a requirement and often a scenario inside it (`... -- scenario "<title>"`). Both halves rot in the same rename, and `correct-the-documents-against-the-tree` corrected and checked only the requirement half, so many citations under `.github/tests` (and one in `.github/workflows/drift.yml`) name a scenario no specification under `openspec/specs/` holds (thirty on 2026-09-13). Several titles were reworded rather than renamed, so each replacement needs a reading, not a substitution.

The `drift.yml` citation is the inverse case: it cites *"Drift in one stack does not resolve another's report"* while `openspec/specs/iac-cicd-pipeline/spec.md` still titles it *"Drift in one environment does not resolve another's report"*. `rename-terraform-environments-to-stacks` left the specification side alone because a `MODIFIED` requirement replaces its block whole and `openspec validate` reads a renamed scenario as dropped; renaming one needs a mechanism, not an edit.

Do the check with the sweep: `.github/tests/test_the_retired_requirement_names_are_gone.py` already reads every tracked file, flattens wrapped comments and derives its subject from committed specifications. The scenario predicate is its inverse (a cited title **not** among live scenario titles), so it needs its own reader; measure its false-positive rate first, since any quoted phrase looks like a citation.

**Kind**: `change`. It needs a decided mechanism for renaming a scenario through a `MODIFIED` delta, and the `drift.yml` case owes an `iac-cicd-pipeline` delta.

**Waits on**: `nothing`.

**Design with**: `rename-the-requirements-that-read-narrower-than-they-are`.

**Recorded by**: `correct-the-documents-against-the-tree` (2026-09-13), which swept requirement names; the scenario half is a different predicate over a different set and would have doubled its diff with a second unchecked sweep.

**Re-checked**: 2026-10-19.

## `separate-history-from-rationale-in-source-comments`

Source comments mix three kinds of text with no way to tell them apart: why the code is shaped this way (keep), what a past change did (git and the archive own it), and TODOs whose condition has passed (dead and misleading). A pass should leave only the first. The ratio is worst in `ansible/roles/tailscale/tasks/main.yml` (140 comment lines of 197 non-blank), and it has grown rather than held.

**Must not undo**: load-bearing rationale such as the tailnet `POLARITY` notes in `ansible/roles/tailscale/tasks/main.yml` and the `ghcr_pull_*` tolerated/not-tolerated block in `ansible/roles/deploy_user/tasks/main.yml`. The two instances this entry once named (`terraform/stacks/main-production/ssh_key.tf`'s `moved` block, `main.tf`'s `delete_protection` parenthetical) were taken as fixes on 2026-09-15; the pass needs instances of its own. The citation-form decision it waited on was delivered by `decide-archived-change-reference-policy` (archived 2026-09-07).

**Kind**: `change`. A repository-wide pass is too large for one review sitting, and the rule for what a comment may hold needs the operator's decision and a home in `AGENTS.md`.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: unknown; it was unblocked by `decide-archived-change-reference-policy`, which changed citation form only and deliberately left the surrounding prose alone.

**Re-checked**: 2026-10-19.

## `record-the-hostname-scheme`

Public names follow `<service>.<server>.BASE_DOMAIN` (`<app>.main-production.BASE_DOMAIN`, `<app>.main-staging.BASE_DOMAIN`), resolved by one wildcard `A` record per server, with a short alias directly under `BASE_DOMAIN` as a record of its own. The rule was decided in the `orders-ops` repository, whose `deploy-orders-ops-to-staging` handoff names `docs/naming-conventions.md` here as its home, but that file names servers, stacks, keys and the OS hostname and not public names.

It owes: the rule; where the base domain comes from; and how it composes with the `<company>` segment and the stack name that file already rules on, since a company-owned base domain is a naming axis `<company>` does not yet cover. `docs/bootstrap-a-new-host.md` §4.4 should then point at the rule rather than restate the wildcard shape.

**Kind**: `change`. Documentation only, but how a company-owned base domain composes with `<company>` is a naming decision the operator has not made.

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `expose-staging-on-the-web` (2026-09-14), which wrote down the DNS the scheme produces but not the scheme.

**Re-checked**: 2026-10-19.

## `announce-a-rebuild-to-the-applications-that-hold-databases`

*The Host Carries an Operator-Declared Maintenance Window for the Shared Instance* (`openspec/specs/iac-host-configuration/spec.md`) lets each application's probe answer `window-open` from a declaration file under `/var/lib/platform-maintenance/` during a destructive act on the shared instance. It cannot announce a rebuild, because the declaration lives on the host the rebuild destroys: the probe gets no answer, which is exactly the state of a host that has fallen over, the thing the probe exists to tell apart.

The change owes a decision on who the audience is, then where such a declaration lives: off the host (a repository variable, status page, or field the application's deploy reads) survives the rebuild but needs a second mechanism; on the host but restored early closes only part of the window; out of band to operators is not machine-readable. Today `docs/runbook-rebuild.md` phase 1 tells the operator to tell application owners by hand, and points here.

**Kind**: `change`. It needs the operator's decision on audience and location, and owes an `iac-host-configuration` delta.

**Waits on**: a second application holding a database in the shared instance, or a rebuild performed by someone other than the operator who would announce it. Today only `orders-ops` holds one.

**Design with**: `nothing`.

**Recorded by**: planning `write-and-rehearse-the-rebuild-runbook` (2026-09-16), which found the gap; it was not a blocker on the runbook because a rebuild is operator-initiated and can be announced by hand.

**Re-checked**: 2026-10-19.

## `make-a-rotated-secret-reach-its-inline-config`

A secret interpolated into an inline `configs:` block in `platform/docker-compose.yml` does not reach the container on a deploy, and the deploy reports success. `platform.config-checksum` (from `apply-shipped-config-on-deploy`) is a literal computed from the committed config text; a secret moves only the rendered config, so `docker compose up -d --wait` sees an identical service and leaves it alone. Affected today: `DEADMANSWITCH_URL` and `SLACK_WEBHOOK_URL` in `alertmanager_config`, i.e. the dead-man's-switch and alert routing, both of which fail silently. Measured on staging 2026-09-16: Alertmanager still pinged the old URL eleven hours after a green deploy. Rotating the Slack webhook does not take effect on a deploy; if the old one is revoked, alert delivery is dead while every check stays green.

Options: force-recreate the services whose configs interpolate a secret; compute the checksum over the rendered config at deploy time (ends the static check, catches both cases); or a post-deploy probe asserting the running config carries the shipped value. It is a property of a running container, so neither `.github/tests` nor Molecule can hold the test. Until it lands, a rotation is followed by recreating the service by hand and confirming the value arrived. A non-inline `environment:` value is recreated correctly; Grafana ignoring a delivered password is `make-a-rotated-grafana-password-reach-grafana`, not this.

**Kind**: `change`. It needs the operator to choose among the three options and owes a delta to the platform deploy requirements.

**Waits on**: `nothing`.

**Design with**: `make-a-rotated-grafana-password-reach-grafana`, `perform-the-stack-separation-check-that-was-never-run`, `verify-at-deploy-time-that-what-shipped-is-what-runs`.

**Recorded by**: found 2026-09-16 while closing `give-staging-its-own-dead-mans-switch-check`, by checking whether the fix had landed rather than reading the deploy's result; it is a separate defect in the deploy mechanism.

**Re-checked**: 2026-10-19.

## `perform-the-stack-separation-check-that-was-never-run`

`docs/bootstrap-a-new-host.md` §7.5 asks the operator to confirm the two stacks are genuinely separate: the two Grafanas must want different passwords, and a test alert must arrive in one channel rather than both. It had never been performed; when it was (2026-09-16), production and staging shared the Slack webhook (alerts from both landed in staging's workspace), the dead-man's-switch URL and the Grafana admin password. All three are now separated (`apply-the-per-stack-alert-routing`, 2026-09-16; `give-each-stack-its-own-grafana-password`, 2026-09-17, which rotated staging's value and deliberately left production's).

What remains is a repeatable check. These are GitHub Environment secrets, invisible to `.github/tests`, Molecule and review; the only defence is a runbook step read once. A probe comparing the two running hosts is the workable shape, and it needs a home. It must compare what the hosts **accept**, not only what they hold: on 2026-09-17 Grafana held a new password and accepted the old one.

**Kind**: `change`. It needs a decision on where a cross-host probe lives and whether the separation is worth asserting repeatably, and owes a delta for it.

**Waits on**: `nothing`.

**Design with**: `make-a-rotated-secret-reach-its-inline-config`, `make-a-rotated-grafana-password-reach-grafana`, `label-each-stack-s-alerts-with-the-stack-they-came-from`.

**Recorded by**: the rebuild rehearsal's pre-state capture (2026-09-16), during `write-and-rehearse-the-rebuild-runbook`; the three values were fixed as fixes, and the repeatable check is a change of its own.

**Re-checked**: 2026-10-19.

## `decide-whether-the-data-volume-should-survive-a-server-toggle`

Each stack couples its data volume to its server (`count = var.volume_enabled && var.server_enabled ? 1 : 0` in `terraform/stacks/*/main.tf`), so `server_enabled = false` discards the volume, which has no Hetzner backup. `add-prod-data-volume`'s design accepted this deliberately. The 2026-09-16 rehearsal showed the apply destroys the volume **first**, before the server. Decoupling would give the volume its own `location` and gate it on `volume_enabled` alone; the cost is not money but the toggle's meaning (`server_enabled` is "the way to stop paying for staging without deleting its configuration"), since a surviving volume makes "off" neither free nor clean.

It is not worth doing while *No Store on This Host Holds Data Requiring Backup* (`openspec/specs/iac-safety-hardening/spec.md`) holds: nothing on the volume is durable. **Must not undo**: when durable data lands on a host, decoupling and a real backup are owed together; decoupling alone looks like protection while protecting against nothing. The divergence that requirement records, `orders-ops`'s own PostgreSQL on production, which nothing backs up, is the first candidate to make the trigger real.

**Kind**: `change`. It owes an `iac-safety-hardening`/stack delta and the operator's decision on the toggle's meaning, delivered with a backup.

**Waits on**: durable data landing on a stack's data volume, i.e. a store there that *No Store on This Host Holds Data Requiring Backup* cannot classify as needing no backup.

**Design with**: `exercise-the-volume-server-coupling-against-live-state`.

**Recorded by**: a question during `write-and-rehearse-the-rebuild-runbook`'s rehearsal (2026-09-16), once the destroy had shown what the coupling does; deliberately deferred as not yet worth doing.

**Re-checked**: 2026-10-19.

## `make-a-rotated-grafana-password-reach-grafana`

Rotating `PLATFORM_GRAFANA_ADMIN_PASSWORD` recreates `platform-grafana-1` with the new value and leaves Grafana accepting the old one: `admin_password` is read only when Grafana's database is created, and that database lives on the data volume (`/mnt/main/grafana`). Measured on staging 2026-09-17: deploy green, container `Recreated`, new value in the environment, `/api/user` answered `401`. Every signal is green; the shape that hurts is a rotation performed because a credential leaked, which leaves the leaked credential working. Until it lands, the host step in `docs/bootstrap-a-new-host.md` Appendix A (`grafana cli admin reset-admin-password`) completes a rotation, and the operator confirms a login rather than reading the deploy.

The change owes either a deploy that reconciles the service with the shipped value (smaller for Grafana alone; belongs where the deploy can hold a credential, i.e. `app-deploy`) or a post-deploy probe asserting the running service accepts what was shipped. The probe generalises to every credential its service stored once: `PLATFORM_POSTGRES_EXPORTER_PASSWORD` (typed into Postgres at §7.5) and `PLATFORM_POSTGRES_USER`/`_PASSWORD` (used by `initdb` at first boot). It has the same home problem as its siblings: a property of a running host.

**Kind**: `change`. It needs the operator to choose reconcile or probe, and owes a platform deploy delta.

**Waits on**: `nothing`.

**Design with**: `make-a-rotated-secret-reach-its-inline-config`, `perform-the-stack-separation-check-that-was-never-run`, `verify-at-deploy-time-that-what-shipped-is-what-runs`.

**Recorded by**: `give-each-stack-its-own-grafana-password` (2026-09-17), which rotated staging's password and checked a login; fixing the rotation mechanism was outside that change's scope.

**Re-checked**: 2026-10-19.

## `hold-every-alert-to-naming-what-it-fires-about`

A rule that interpolates `$labels.<L>` into its notification should exclude series where `<L>` is empty. `scope-the-restart-alert-to-real-containers` fixed and asserted this for `ContainerRestartingOrOOMKilled` only (it had sent `Container  is crash-looping or was OOM-killed` eight times). In `platform/docker-compose.yml`, `ApplicationHighErrorRate` (`by (router)`) and `TLSCertificateExpiringSoon` (`by (cn)`) do not exclude the empty value, and an empty `cn` is known to occur (`CertificateExpiryNotObserved` exists to detect `cn=""`).

First measure whether either can reach a notification; that decides whether this includes fixing two rules or only adds the guard against the next rule. The guard belongs in the static suite, generalising the reader in `.github/tests/test_the_container_alert_names_a_container.py`. It must decide what counts as excluding the empty value: `sum by (router)` keeps an empty `router` as its own group, so accepting `by (<L>)` as the guard would pass both rules while proving nothing.

**Kind**: `change`. Generalising beyond container rules owes a delta to *Metrics-Based Alerting Covers Host and Service Health* (`openspec/specs/iac-platform-services/spec.md`).

**Waits on**: `nothing`.

**Design with**: `nothing`.

**Recorded by**: `scope-the-restart-alert-to-real-containers` (2026-09-17), whose subject is one rule; this is a property of all of them.

**Re-checked**: 2026-10-19.

## `share-the-static-suites-test-arrangement`

The `.github/tests` suite has no shared test arrangement: several modules import helpers from `test_ci_configuration.py`, a test module, so a module that is rewritten for its own reasons breaks others, and a test author has nowhere to take a builder from. Give the suite a support module the tests import from, move those helpers into it, and record its location under *Testing* in `AGENTS.md` as the suite's shared test arrangement. Molecule and `terraform test` have none either; say whether either needs one.

**Kind**: `change`. It decides the project's testing strategy (where shared arrangement lives), which is the operator's decision.

**Waits on**: `nothing`.

**Design with**: `hold-the-whole-static-suite-to-its-own-constraints`, which also changes how that suite's modules read the tree.

**Recorded by**: `sync-ai-toolkit-workflow-v10`, which recorded the shared test arrangement as none when taking ai-toolkit's development-workflow v10; creating one is not a sync.

**Re-checked**: 2026-10-19.
