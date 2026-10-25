---
kind: change
waits_on: []
design_with: []
recorded_by: scope-the-restart-alert-to-real-containers
re_checked: 2026-10-19
---

A rule that interpolates `$labels.<L>` into its notification should exclude series where `<L>` is empty. `scope-the-restart-alert-to-real-containers` fixed and asserted this for `ContainerRestartingOrOOMKilled` only (it had sent `Container  is crash-looping or was OOM-killed` eight times). In `platform/docker-compose.yml`, `ApplicationHighErrorRate` (`by (router)`) and `TLSCertificateExpiringSoon` (`by (cn)`) do not exclude the empty value, and an empty `cn` is known to occur (`CertificateExpiryNotObserved` exists to detect `cn=""`).

First measure whether either can reach a notification; that decides whether this includes fixing two rules or only adds the guard against the next rule. The guard belongs in the static suite, generalising the reader in `.github/tests/test_the_container_alert_names_a_container.py`. It must decide what counts as excluding the empty value: `sum by (router)` keeps an empty `router` as its own group, so accepting `by (<L>)` as the guard would pass both rules while proving nothing.

- **Kind**: Generalising beyond container rules owes a delta to *Metrics-Based Alerting Covers Host and Service Health* (`openspec/specs/iac-platform-services/spec.md`).
- **Recorded by**: `scope-the-restart-alert-to-real-containers` (2026-09-17), whose subject is one rule; this is a property of all of them.
