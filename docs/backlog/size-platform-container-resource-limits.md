---
kind: change
waits_on: []
waits_for: "the operator's judgement that the \"Container health\" dashboard holds enough history to show real steady state and peak; metrics have accumulated since `add-platform-monitoring`."
design_with: []
recorded_by: unknown
re_checked: 2026-10-19
---

No service in `platform/docker-compose.yml` declares a memory or CPU limit on a `cx33`, so one container can starve the host; `ContainerRestartingOrOOMKilled` only alerts on the consequence. Size limits from observed steady state and peak (`container_memory_usage_bytes` by container, on the "Container health" dashboard), not by guess: limits picked without evidence cause the outage they were meant to prevent.

- **Kind**: It needs sizing decisions from measured data across every platform service.
- **Recorded by**: `unknown`.
