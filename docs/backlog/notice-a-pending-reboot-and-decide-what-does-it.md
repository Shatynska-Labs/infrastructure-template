---
kind: change
waits_on: []
design_with: [alert-on-a-scheduled-units-own-output, close-public-ssh-and-manage-sshd-explicitly, hold-every-platform-deploy-for-the-length-of-a-window]
recorded_by: "the operator"
re_checked: 2026-10-19
---

Both hosts carried `/var/run/reboot-required` (kernel, `linux-base`, `libc6`) from 2026-09-10 to 2026-09-16 with nothing reporting it, found only when they were rebooted by hand. `unattended-upgrades` is installed and enabled by the image, not by `hardening`, and every `Automatic-Reboot` line in `/etc/apt/apt.conf.d/` is commented out, so patches install and do not take effect for an unbounded time while the host looks healthy.

It is two changes under one name. *Noticing* is a metric and an alert: node-exporter's textfile collector is not enabled (`platform/docker-compose.yml`'s `node-exporter` has no `--collector.textfile.directory` and no mount for one), so it needs a `platform/` change for the collector and rule plus an `ansible/` unit that writes the metric, reaching the host through two pipelines. *Deciding what reboots* is a `hardening` change: an `Automatic-Reboot` policy with a window, or a stated operator ritual. Rebooting is known-safe: both hosts rebooted cleanly on 2026-09-16 in about forty seconds (containers back on `restart: unless-stopped`, `/mnt/main` mounted before Docker, swap, `ufw`, `tailscaled` up). A reboot is an outage of every application on the host, so an automatic production policy is a scheduled outage; `hold-every-platform-deploy-for-the-length-of-a-window` is what would keep a deploy out of it.

- **Kind**: It needs the operator's reboot-policy decision, spans two pipelines, and owes deltas to `iac-host-configuration` (unattended upgrades owned by `hardening`) and `iac-platform-services` (the collector and alert).
- **Design with**: `alert-on-a-scheduled-units-own-output` (the same unenabled textfile collector: flag, directory, ownership and mount built once for both), `close-public-ssh-and-manage-sshd-explicitly` (the other half of the image-default `hardening` gap), `hold-every-platform-deploy-for-the-length-of-a-window`.
- **Recorded by**: the operator and a session on 2026-09-16, from the hosts after rebooting them by hand; not by a change.
