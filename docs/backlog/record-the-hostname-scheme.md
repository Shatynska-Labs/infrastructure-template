---
kind: change
waits_on: []
design_with: []
recorded_by: expose-staging-on-the-web
re_checked: 2026-10-19
---

Public names follow `<service>.<server>.BASE_DOMAIN` (`<app>.main-production.BASE_DOMAIN`, `<app>.main-staging.BASE_DOMAIN`), resolved by one wildcard `A` record per server, with a short alias directly under `BASE_DOMAIN` as a record of its own. The rule was decided in the `orders-ops` repository, whose `deploy-orders-ops-to-staging` handoff names `docs/naming-conventions.md` here as its home, but that file names servers, stacks, keys and the OS hostname and not public names.

It owes: the rule; where the base domain comes from; and how it composes with the `<company>` segment and the stack name that file already rules on, since a company-owned base domain is a naming axis `<company>` does not yet cover. `docs/bootstrap-a-new-host.md` §4.4 should then point at the rule rather than restate the wildcard shape.

- **Kind**: Documentation only, but how a company-owned base domain composes with `<company>` is a naming decision the operator has not made.
- **Recorded by**: `expose-staging-on-the-web` (2026-09-14), which wrote down the DNS the scheme produces but not the scheme.
