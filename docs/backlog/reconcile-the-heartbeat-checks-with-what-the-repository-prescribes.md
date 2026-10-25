---
kind: fix
waits_on: []
design_with: []
recorded_by: correct-the-documents-against-the-tree
re_checked: 2026-10-19
---

What remains is the slug rule, stated where an operator meets it (`docs/bootstrap-a-new-host.md` §7.1 and Appendix A). The periodic-job checks (`infrastructure-drift`, `infrastructure-pre-commit-autoupdate`, `<inventory_hostname>-prune-host-images`) are pinged as `hc-ping.com/<ping key>/<slug>?create=1` by `drift.yml`, `pre-commit-autoupdate.yml` and the `image_prune` role. Re-slugging one at the observer therefore fails silently: the next ping creates a second check under the old slug, and the renamed one goes quiet a period later. Their display names can be changed freely. The per-stack Alertmanager checks are reached only by their own ping URL (`PLATFORM_DEADMANSWITCH_URL`), so renaming one costs nothing. Appendix A's *Renaming a server afterwards* covers the server-rename case, but §7.1 does not tell a reader which checks are free to rename.

It must not undo the following: never re-slug a periodic-job check to tidy the list. The prune slugs are templated from `inventory_hostname`, so changing one at the observer alone puts it permanently out of step with what the host pings.

The rest of the original entry is done. §7.1 now prescribes `<stack>-alertmanager` checks. Appendix A records their period of 5 minutes and grace of 2 minutes, with the reason (an alarm after about seven minutes of silence).

- **Kind**: It is documentation only, with no decision outstanding.
- **Recorded by**: `correct-the-documents-against-the-tree`. Its operator step looked for a check that the observer did not hold under the prescribed name, and the observer's settings lie outside that change's documents.
