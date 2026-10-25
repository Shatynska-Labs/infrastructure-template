---
kind: change
waits_on: []
design_with: []
recorded_by: provision-orders-ops-database-in-the-shared-instance
re_checked: 2026-10-19
---

*Single Shared PostgreSQL Instance, Per-Application Databases* (`openspec/specs/iac-platform-services/spec.md`) obliges this repository to automate two things: how an application's database and role are provisioned in the shared instance, and how the role's password reaches the application. The trigger fired on 2026-09-13, when `orders-ops` got a database on staging, and production followed. Both were provisioned by hand from the recipe in `docs/onboard-an-application.md`. The requirement carries a paragraph, "One divergence is stated rather than hidden, as of 2026-09-13", saying the obligation stays unmet until this mechanism lands.

A design must answer:

- Where is the password generated, and how does it reach the application's deploy without a command line, a log line or a file in this repository?
- What happens when the secret name is already taken? Production's `orders-ops` Environment already held `POSTGRES_PASSWORD`, so the shared-instance password became `SHARED_POSTGRES_PASSWORD`, and the mechanism must handle that collision for every application.
- A re-run is a rotation, which the application picks up only on its next deploy.
- How is a database re-provisioned after a rebuild or a store reset?
- Each host's credential must be independent, so that one leaked credential reaches one host.

- **Kind**: It owes a MODIFIED delta on *Single Shared PostgreSQL Instance, Per-Application Databases* that replaces the divergence paragraph, an update to the recipe in `docs/onboard-an-application.md`, and a design of the credential path.
- **Recorded by**: `provision-orders-ops-database-in-the-shared-instance` (archived 2026-09-15). It provisioned by hand because a mechanism fitted to one consumer would be found wrong by the next.
