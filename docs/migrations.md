# Database Migrations

Schema is managed by [Liquibase](https://www.liquibase.org/) via
[`src/main/resources/db/changelog/db.changelog-master.xml`](../src/main/resources/db/changelog/db.changelog-master.xml).
Hibernate runs with `ddl-auto=validate` — it never creates or modifies tables.
**Only Liquibase changes the schema.**

## Local development

Migrations run automatically when the app starts:

```bash
# Start the database
docker compose up db

# Run the app — Liquibase applies pending changesets automatically
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

The dev profile reads the database URL from `${DB_NAME:twitter_clone}` and the
root password from `${DB_ROOT_PASS:root_pass}`. The defaults match what
`docker compose up db` creates, so no environment variables are required for
the standard setup.

To use a different database or password, export the matching variable before
starting the app:

```bash
# bash / zsh
export DB_NAME=moo
export DB_ROOT_PASS=my_root_pass
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

```powershell
# PowerShell (Windows)
$env:DB_NAME="moo"
$env:DB_ROOT_PASS="my_root_pass"
./mvnw spring-boot:run "-Dspring-boot.run.profiles=dev"
```

## Applying migrations manually

Useful for CI pipelines, scripted deploys, or previewing changes before an
app start:

```bash
./mvnw liquibase:update \
  -Dliquibase.url="jdbc:mysql://localhost:3306/twitter_clone" \
  -Dliquibase.username=root \
  -Dliquibase.password=root_pass

# Apply only the next N changesets (useful for staged rollouts)
./mvnw liquibase:update-count -Dliquibase.count=1 \
  -Dliquibase.url="jdbc:mysql://localhost:3306/twitter_clone" \
  -Dliquibase.username=root \
  -Dliquibase.password=root_pass
```

## Baselining an existing (pre-Liquibase) database

If a database was created by an older `ddl-auto=update` version that predates
this Liquibase setup, baseline it before running migrations. Baselining marks
the existing changesets as already applied without executing them, so
Liquibase knows where to start applying future changes.

```bash
# Mark all existing changesets as already applied without executing them
./mvnw liquibase:changelog-sync \
  -Dliquibase.url="jdbc:mysql://localhost:3306/legacy_db" \
  -Dliquibase.username=root \
  -Dliquibase.password=root_pass
```

Prefer in-place migration: changesets `002a-add-posts-created-at-column`,
`002a-backfill-posts-created-at`, and `002a-add-posts-created-at-not-null`
provide a safe, retry-friendly migration path for older schemas that have
`posts.timestamp` but not `posts.created_at`. If you prefer to baseline
instead of migrate, run the backfill manually:

```sql
UPDATE posts SET created_at = `timestamp` WHERE created_at IS NULL;
ALTER TABLE posts MODIFY created_at DATETIME(6) NOT NULL;
```

## Production

Production applies pending changesets on startup
(`spring.liquibase.enabled=true`). For zero-downtime deployments, run
migrations as a pre-deploy step before rolling out the new app version,
so failures are caught before any new pods receive traffic:

```bash
./mvnw liquibase:update \
  -Dliquibase.url="${MYSQL_DB_URL}" \
  -Dliquibase.username="${MYSQL_DB_UNAME}" \
  -Dliquibase.password="${MYSQL_DB_PASSWD}"
```

Then roll out the new app version. If a migration needs to be rolled back,
use `liquibase:rollback -Dliquibase.rollbackCount=1` with the same connection
parameters.

See
[`db.changelog-master.xml`](../src/main/resources/db/changelog/db.changelog-master.xml)
for the ordered list of changesets.