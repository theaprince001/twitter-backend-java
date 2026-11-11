# Database Migrations

Schema is managed by [Liquibase](https://www.liquibase.org/) via
[`src/main/resources/db/changelog/db.changelog-master.xml`](../src/main/resources/db/changelog/db.changelog-master.xml).
Hibernate runs with `ddl-auto=validate` — it never creates or modifies tables.
**Only Liquibase changes the schema.**

Liquibase applies pending changesets automatically when the application starts.
No manual migration step is required for local development or production — the
application applies migrations before Hibernate's `ddl-auto=validate` check runs.

## Local development

Set up the environment file and start the database:

```bash
# 1. Copy the example environment file
cp .env.example .env

# 2. Set nonempty values in .env for all required Compose variables
#    (see docker-compose.base.yml for the full list):
#    DB_NAME=twitter_clone
#    DB_USER=moo_user
#    DB_PASS=moo_pass
#    DB_ROOT_PASS=root_pass
#    JWT_SECRET=<at least 32 bytes of randomness>

# 3. Start the database in detached mode so the command returns to the shell
docker compose up -d db
```

Maven does not read `.env` — export every value the application needs.
The dev profile reads the database URL from `${DB_NAME:twitter_clone}` and the
root password from `${DB_ROOT_PASS:root_pass}`, and it requires `JWT_SECRET`
to be set (root `application.properties` uses `${JWT_SECRET}` with no default):

```bash
# bash / zsh
export DB_NAME=twitter_clone
export DB_ROOT_PASS=root_pass
export JWT_SECRET="$(openssl rand -hex 32)"
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

```powershell
# PowerShell (Windows)
$env:DB_NAME="twitter_clone"
$env:DB_ROOT_PASS="root_pass"
$env:JWT_SECRET="<at least 32 bytes of randomness>"
./mvnw spring-boot:run "-Dspring-boot.run.profiles=dev"
```

## Upgrading an existing (pre-Liquibase) database

If a database was created by an older `ddl-auto=update` build that predates
this Liquibase setup, the schema may not match what the changelog creates.
Select the target database explicitly before starting the upgrade. The dev
profile defaults to `twitter_clone` and enables `createDatabaseIfNotExist=true`,
so a legacy setup that used a different name (for example `twitter`) would be
silently ignored — the app would create a fresh empty `twitter_clone` and
upgrade that, leaving the real legacy data untouched. Point the app at the
existing database by exporting `DB_NAME` with the legacy name:

```bash
# bash / zsh — target a legacy database named "twitter"
export DB_NAME=twitter
export DB_ROOT_PASS=root_pass
export JWT_SECRET="$(openssl rand -hex 32)"
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

```powershell
# PowerShell (Windows) — target a legacy database named "twitter"
$env:DB_NAME="twitter"
$env:DB_ROOT_PASS="root_pass"
$env:JWT_SECRET="<at least 32 bytes of randomness>"
./mvnw spring-boot:run "-Dspring-boot.run.profiles=dev"
```

Before the migration runs, verify the startup log shows the intended database:

```text
Database JDBC URL [jdbc:mysql://localhost:3306/twitter?...]
Default catalog/schema: twitter/undefined
```

If `Default catalog/schema` shows `twitter_clone` when you expected `twitter`,
stop the app and re-check the exported `DB_NAME`.

The `posts.created_at` migration is provided in-place by three guarded
changesets in `001-initial-schema.xml`:

- `002a-add-posts-created-at-column` — adds `created_at` if it is missing
- `002a-backfill-posts-created-at` — copies `timestamp` into `created_at`
  when a legacy `timestamp` column is present
- `002a-add-posts-created-at-not-null` — enforces `NOT NULL` once the column
  is populated

Each step is independently guarded, so a partial failure (e.g. the backfill
fails due to a lock timeout) is retried on the next startup instead of being
silently skipped.

Do **not** run `liquibase:changelog-sync` against a legacy database without
verifying that every changeset's schema change already exists. `changelog-sync`
marks each changeset as applied without executing it — if an index or
column from the changelog is missing on the legacy schema, that change is
silently skipped and will never be applied on future startups.

## Production

Production applies pending changesets on application startup
(`spring.liquibase.enabled=true` in `application.properties`). What happens
when a migration fails depends on the deployment topology:

- **Rolling / blue-green deployments** (two app instances, traffic routed
  between them): the previous instance keeps serving traffic while the new
  instance applies migrations. If a migration fails, only the new instance is
  affected, and the old one continues serving.
- **Single-container Compose** (this repository's default deployment):
  `docker compose up` stops and recreates the app container when its image
  changes. A migration failure at startup leaves no version serving traffic
  until the failure is resolved and a working image is redeployed.

For zero-downtime migrations, use a multi-instance setup where the previous
version keeps running while the new version applies changes.

Rollback is **not a supported recovery path**. The current changelog does
not define `<rollback>` blocks, so `liquibase:rollback` can only reverse
changesets via Liquibase's auto-generated inverse operations — which for
`010-create-refresh-tokens-table` means dropping the table and all stored
refresh tokens. If a migration needs to be reversed, deploy a new
forward-only changeset that compensates for the previous one.