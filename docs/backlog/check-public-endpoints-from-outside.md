---
kind: change
waits_on: []
waits_for: "the operator's choice of an external uptime service."
design_with: []
recorded_by: unknown
re_checked: 2026-10-19
---

Nothing checks from outside a host that a public hostname resolves and answers on 443. Two failures stay unseen. The first is a cloud-firewall change blocking 443 made at the console: the drift workflow reports only what Terraform manages, and UFW is not in 443's path (see `say-what-the-host-firewall-actually-gates`). The second is a DNS failure with no edit behind it, since both zones are served by third-party nameservers (`docs/bootstrap-a-new-host.md` §4.4). An expiring certificate is already covered by the `TLSCertificateExpiringSoon` rule in `platform/docker-compose.yml`. A hostname served no certificate at all produces no `traefik_tls_certs_not_after` series, so that case is not covered.

A host-local `blackbox-exporter` cannot serve this purpose: a probe to the host's own public address is delivered locally and never crosses the Hetzner cloud firewall. What is left is an external uptime service with a check per hostname, independent of the host as the dead-man's switch is. healthchecks.io only receives inbound pings, so this is probably a second vendor account.

- **Kind**: It needs the operator's decision on an external uptime vendor and its cost.
- **Recorded by**: `unknown`. It was recorded because the monitoring stack watches the host and not the customer's path to it, and it was narrowed by `alert-on-certificate-expiry`, which delivered the certificate-expiry half.
