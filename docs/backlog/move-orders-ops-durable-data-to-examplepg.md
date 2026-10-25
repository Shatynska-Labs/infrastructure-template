---
kind: change
waits_on: [classify-orders-ops-production-data-for-the-shared-instance]
waits_for: "`classify-orders-ops-production-data-for-the-shared-instance` merging, and then the host check above showing that the `orders-ops` repository has removed its private PostgreSQL."
design_with: [classify-orders-ops-production-data-for-the-shared-instance]
recorded_by: unknown
re_checked: 2026-10-19
---

*No Store on This Host Holds Data Requiring Backup* (`openspec/specs/iac-safety-hardening/spec.md`) states one divergence, as of 2026-09-08: on production, `orders-ops` keeps durable data in a PostgreSQL container of its own on its `app_db` network, and nothing backs it up. `docs/runbook-rebuild.md` repeats this. Nothing else tracks the divergence. The operator decided on 2026-09-14 to resolve it in two stages. First, the whole database moves into production's shared instance. That includes the `procrastinate_*` queue tables, because the application enqueues inside its domain transaction. Later, the database moves to ExamplePG, which owns its own backups.

Ending the private container takes four steps:

1. Done by `provision-orders-ops-database-in-the-shared-instance`: the role, the database and `SHARED_POSTGRES_PASSWORD` (deliberately not `POSTGRES_PASSWORD`).
2. `classify-orders-ops-production-data-for-the-shared-instance`, in this repository.
3. The `orders-ops` repository's own migration. It is a dump-and-restore, and `DATABASE_URL` is repointed to `orders-ops@postgres:5432/orders-ops`. The application's PostgreSQL service, its volume and its `app_db` network are removed in the same deploy, because while both exist `postgres` names two servers.
4. Here: delete the divergence paragraph and the runbook's matching paragraph.

Gate step 4 on the host, not on a report: on production, `docker ps --filter name=orders-ops` shows no PostgreSQL container, and `docker volume ls` no longer lists `orders-ops_orders_ops_pgdata`.

For the later move to ExamplePG, use the session pooler `db-pooler.example.net:5432` with user `postgres.<ref>`, which answered `pg_isready` from staging's `platform_edge` on 2026-09-14. The pooler is needed for three reasons: the direct connection is IPv6-only while `platform_edge` has IPv6 disabled, the transaction pooler on port 6543 cannot carry the worker's `LISTEN/NOTIFY`, and the Free plan has no backups.

- **Kind**: It owes a MODIFIED delta on *No Store on This Host Holds Data Requiring Backup* that removes the divergence. The migration itself is the `orders-ops` repository's work.
- **Recorded by**: `unknown`. It was recorded because the specification names the divergence and nothing else tracked it, and the migration belongs to the `orders-ops` repository, outside this one's authority.
