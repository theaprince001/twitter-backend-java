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

# 2. Set nonempty values in .env for the required Compose variables
#    (see docker-compose.base.yml for the full list):
#    DB_NAME=twitter_clone
#    DB_USER=moo_user
#    DB_PASS=moo_pass
#    DB_ROOT_PASS=root_pass

# 3. Start the database
docker compose up db
```

Then run the application. Liquibase applies pending changesets on startup:

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

The dev profile reads the database URL from `${DB_NAME:twitter_clone}` and the
root password from `${DB_ROOT_PASS:root_pass}`. If you set different values in
`.env` for Compose, export the same values for the application so both connect
to the same database:

```bash
# bash / zsh
export DB_NAME=twitter_clone
export DB_ROOT_PASS=root_pass
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

```powershell
# PowerShell (Windows)
$env:DB_NAME="twitter_clone"
$env:DB_ROOT_PASS="root_pass"
./mvnw spring-boot:run "-Dspring-boot.run.profiles=dev"
```

## Upgrading an existing (pre-Liquibase) database

If a database was created by an older `ddl-auto=update` build that predates
this Liquibase setup, the schema may not match what the changelog creates.
The `posts.created_at` migration is provided in-place by three guarded
changesets in `001-initial-schema.xml`:

- `002a-add-posts-created-at-column` — adds `created_at` if it is missing
- `002a-backfill-posts-created-at` — copies `timestamp` into `created_at`
  when a legacy `timestamp` column is present
- `002a-add-posts-created-at-not-null` — enforces `NOT NULL` once the column
  is populated

Each step is independently guarded, so a partial failure (e.g. the backfill
fails due to a lock timeout) is retried on the next startup instead of being
silently skipped. Just start the application against the legacy database:

```bash
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

Liquibase runs the three `002a-*` changesets in order, preserving existing
`timestamp` data.

Do **not** run `liquibase:changelog-sync` against a legacy database without
verifying that every changeset's schema change already exists. `changelog-sync`
marks each changeset as applied without executing it — if an index or
column from the changelog is missing on the legacy schema, that change is
silently skipped and will never be applied on future startups.

## Production

Production applies pending changesets on application startup
(`spring.liquibase.enabled=true` in `application.properties`). The recommended
deployment flow:

1. Deploy the new application version.
2. Liquibase runs before the app becomes ready. If a changeset fails, the app
   does not start, and the previous version continues serving traffic.
3. Verify `DATABASECHANGELOG` shows the expected changesets applied.
4. For zero-downtime deployments, deploy to a single canary instance first,
   verify the schema, then roll out the remaining instances.

Rollback is **not a supported recovery path**. The current changelog does
not define `<rollback>` blocks, so `liquibase:rollback` can only reverse
changesets via Liquibase's auto-generated inverse operations — which for
`010-create-refresh-tokens-table` means dropping the table and all stored
refresh tokens. If a migration needs to be reversed, deploy a new
forward-only changeset that compensates for the previous one.