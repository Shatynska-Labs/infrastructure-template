---
kind: change
waits_on: []
design_with: []
recorded_by: unknown
re_checked: 2026-10-19
---

Logs are read by `docker logs` over SSH, per container, and are lost when a container is recreated, which every deploy does; alerts say *that* a container restarted while the reason is in the log that went away. Low priority for a host running one application, and the first thing missed when it runs several.

Loki with an Alloy (or Promtail) collector reading the Docker socket is the stack-native answer: it joins `platform_monitoring`, Grafana's datasource provisioning pattern applies, retention is bounded as Prometheus's is, and it stores on the data volume under a `platform_data_volume_subdirs` entry. Its prerequisite is delivered: the `docker` role bounds json-file logs at the daemon (`bound-host-log-growth-and-add-swap`), and the collector reads the same files. Traefik's access log is on (`--accesslog=true`, since 2026-09-13), which now shortens Traefik's own `docker logs` history.

- **Kind**: It adds platform services, owing an `iac-platform-services` delta, and needs the collector and retention decisions.
- **Recorded by**: `unknown`.
