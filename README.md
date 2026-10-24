# stornuage-db-management

Centralized Liquibase repository that owns the DDL for every StorNūAge service schema. No service runs Liquibase itself at startup. Schema changes ship through this repo and are applied by CI/CD before any service is deployed.

## Why this exists

Previously, DDL lived scattered across three services:
- `nutritionmeter-service/src/main/resources/database/*.sql` (never executed by Flyway)
- `notification-service/src/main/resources/db/migration/V*.sql` (Flyway on startup)
- integration-service shared the nutritionmeter schema with no migrations of its own

That left three problems:
1. **No single source of truth for schema**: ad-hoc SQL files could drift from production.
2. **Startup-time migrations are risky at scale**: at 100k msg/s across N pods, concurrent Liquibase runs rely on `databasechangeloglock`, but any failure can brick the deployment.
3. **Shared schema between services** broke bounded contexts — a nutritionmeter ALTER could crash integration without CI signal.

This repo fixes all three.

## Architecture

- **Three schemas**, one owner each:
  - `integration` — owned by integration-service
  - `nutritionmeter` — owned by nutritionmeter-service
  - `stornuage_notifications` — owned by notification-service
- **One Liquibase job** applies all three in a single transaction-managed run before the services deploy.
- **Services do NOT run Liquibase** — they connect read/write to their own schema with a role that lacks `CREATE`/`ALTER` privileges on tables.
- **`databasechangelog` + `databasechangeloglock` tables** are created by Liquibase automatically; MD5 checksums validate every changeset on every run — any mutation of an already-applied changeset fails the pipeline.

## Repo layout

```
.
├── pom.xml                                  Maven project driving the liquibase-maven-plugin
├── liquibase.properties.template            Rendered by CI with environment-specific secrets
├── src/main/resources/
│   ├── master-changelog.xml                 Entry point, includes 3 per-schema masters
│   └── changesets/
│       ├── integration/
│       │   ├── changelog-master.xml
│       │   ├── 001-create-schema.xml
│       │   ├── 002-container-registry.xml
│       │   ├── 003-item-correction.xml
│       │   ├── 004-item-alias.xml
│       │   ├── 005-idempotency-key.xml
│       │   └── 006-outbox.xml
│       ├── nutritionmeter/
│       │   ├── changelog-master.xml
│       │   ├── 001-create-schema.xml
│       │   ├── 002-container-profile.xml
│       │   ├── 003-item-nutrition.xml
│       │   ├── 004-container-registry.xml
│       │   ├── 005-consumption-event.xml
│       │   ├── 006-item-depletion-forecast.xml
│       │   ├── 007-nutrition-summary.xml
│       │   ├── 008-idempotency-key.xml
│       │   ├── 009-outbox.xml
│       │   └── 010-seed-data.xml           (contexts=dev,staging only)
│       └── notifications/
│           ├── changelog-master.xml
│           ├── 001-create-schema.xml
│           ├── 002-user-device.xml
│           ├── 003-alert.xml
│           ├── 004-processed-signal.xml
│           ├── 005-alert-state-audit.xml
│           ├── 006-notification-delivery.xml
│           ├── 007-idempotency-key.xml
│           └── 008-outbox.xml
└── .github/workflows/
    ├── validate.yml                         PR: validates + applies against ephemeral Postgres
    └── apply.yml                            Push to main: applies to DEV; manual to staging/prod
```

## Running locally

```bash
cp liquibase.properties.template liquibase.properties
# then fill in LB_URL / LB_USER / LB_PASSWORD manually, or export them and run:
sed -e "s|\${LB_URL}|jdbc:postgresql://localhost:5432/stornuage|g" \
    -e "s|\${LB_USER}|postgres|g" \
    -e "s|\${LB_PASSWORD}|postgres|g" \
    liquibase.properties.template > liquibase.properties

mvn liquibase:validate
mvn liquibase:status
mvn liquibase:update -Dliquibase.contexts=dev
```

Seed data for `nutritionmeter` is gated behind `contexts=dev,staging` — production migrations skip it.

## The golden rules

1. **Changesets are immutable once merged.** Never modify a changeset XML after it has been applied in any shared environment. Liquibase validates the MD5 checksum against `databasechangelog.MD5SUM`; a mismatch halts the pipeline.
2. **Always add new changesets.** To change a column, write a new changeset with `addColumn` / `renameColumn` / `dropColumn`. Never edit the original `createTable`.
3. **Every changeset has a `<rollback>` block.** Non-reversible changes (data deletions, column drops) must document the rollback strategy explicitly, even if just a comment.
4. **One logical change per changeset.** Easier to review, easier to roll back.
5. **Author = `stornuage`** for pipeline-owned changesets; a human author tag is fine for one-off hotfixes.
6. **No cross-schema foreign keys.** Each service owns its schema; cross-service relationships are expressed as events (SQS) or internal APIs (HMAC-signed).

## Rollback

```bash
# roll back the last applied changeset
mvn liquibase:rollback -Dliquibase.rollbackCount=1

# roll back everything applied after a tagged release
mvn liquibase:rollback -Dliquibase.rollbackTag=release-2026-10-10
```

Tag releases from CI (`mvn liquibase:tag -Dliquibase.tag=release-<yyyy-mm-dd>`) so prod can reliably roll back to any known-good state.

## CI pipelines

| Trigger | Workflow | What happens |
|---|---|---|
| PR to `main` | `validate.yml` | Spins up ephemeral Postgres, runs `liquibase:validate`, `liquibase:update`, verifies zero pending, and smoke-tests `rollback`. Fails the PR on checksum drift. |
| Push to `main` | `apply.yml` → `apply-dev` | Applies to DEV automatically. |
| `workflow_dispatch` → `staging` | `apply.yml` → `apply-staging` | Prints `updateSQL` preview, then applies. Protected by `environment: staging`. |
| `workflow_dispatch` → `prod` | `apply.yml` → `apply-prod` | Uploads the rendered SQL as an artifact for review, then applies. Protected by `environment: prod` (manual approval). |

## Checksum enforcement — why it matters

Liquibase stores an MD5 hash of each changeset in `databasechangelog.MD5SUM` on first apply. If a developer later edits the XML of an already-applied changeset, the next `liquibase:update` will fail:

```
Validation Failed:
  1 changesets check sum
    changesets/integration/002-container-registry.xml::integration-002-container-registry::stornuage was: 8:abc... but is now: 8:def...
```

This is deliberate. It makes schema drift impossible to sneak into production unnoticed. If you absolutely must mutate an already-applied changeset (e.g., to fix a typo in a comment), run `liquibase:clearCheckSums` locally against a disposable DB — never in a shared environment.
