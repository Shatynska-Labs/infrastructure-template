---
kind: change
waits_on: []
design_with: []
recorded_by: unknown
re_checked: 2026-10-19
---

postgres-exporter once built `DATA_SOURCE_NAME` by interpolating a password into a URL, and a `#` in that password discarded the host (see `alert-on-the-exporter-being-unable-to-read-postgres`). The service now uses `DATA_SOURCE_URI`, `DATA_SOURCE_USER` and `DATA_SOURCE_PASS`. What is still owed is a static check in `.github/tests` that stops the shape from returning.

The rule is: a secret-bearing variable inside a URL, within its own whitespace-delimited word.

- Offences: a scheme URL; a `user:secret@host` authority with no scheme; a query parameter; a path segment of an authority with no scheme; text concatenated onto a secret that is itself a URL.
- Not offences: a value that is only the secret (Alertmanager's `api_url`); a non-secret host; a secret next to an unrelated URL; a filesystem path; Compose's `$$`.

An earlier draft repeated one defect across three review rounds. Each arm must read the quote-stripped text around the secret rather than anchor to the start of the raw word, because quoting inside `configs:` and `--flag=value` inside `command:` are both idiomatic.

Secret names must be read from the `env:` block of `platform-deploy.yml`'s render step, not guessed from keywords: `DEADMANSWITCH_URL` is a bearer credential although nothing in its name says so. The scan must cover `command:`, `entrypoint:`, `labels:`, `healthcheck.test`, `environment:` and the inline `configs:`, since `${ACME_EMAIL}` appears only in `command:`. The check must not be able to pass having read nothing, so it asserts a floor on the number of interpolations it found.

- **Kind**: It owes a requirement stating the rule the check enforces (`iac-platform-deploy-pipeline`), and its earlier draft did not fit one review sitting.
- **Recorded by**: `unknown`. It was split on 2026-09-15 from the fix to postgres-exporter's connection settings, because the check's draft was still wrong after three review rounds.
