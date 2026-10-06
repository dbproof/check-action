# DbProof check

Linters check your SQL. [DbProof](https://dbproof.dev) checks your migrations against production's shape.

On every pull request that changes migrations, this action restores a snapshot of your production schema and its row statistics into a throwaway Postgres, applies the pull request's migrations one at a time with your own migrate command, and reports on the pull request what would fail, lock or drift in production.

- **Failures production would hit:** a column a hotfix already added, `SET NOT NULL` on a column with nulls.
- **Locks on large tables:** an index built without `CONCURRENTLY`, a rewrite of a table with millions of rows.
- **Risks to the running application:** a dropped column, a foreign key without an index.

Works with Flyway, Atlas, Prisma Migrate and Drizzle Kit, on PostgreSQL 13 and later. It never reads a row of your data, and CI never connects to production.

## Usage

Connect your repository at [app.dbproof.dev](https://app.dbproof.dev): DbProof finds your migrations and opens a pull request with this workflow, filled in. By hand, it looks like this:

```yaml
name: DbProof check
on:
  pull_request:
    paths: ["db/migration/**"]
permissions:
  contents: read
  id-token: write # the check proves its repository with GitHub's OIDC token: no secret
jobs:
  migrations:
    name: dbproof/migrations
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: dbproof/check-action@v1
        with:
          project: your-org/your-project
          migrations-dir: db/migration
          migrate-command: >-
            docker run --rm --network host -v "$PWD/db/migration:/flyway/sql"
            flyway/flyway:11-alpine -url="$DBPROOF_CHECK_JDBC_URL"
            -user="$DBPROOF_CHECK_USER" -password="$DBPROOF_CHECK_PASSWORD"
            -locations=filesystem:/flyway/sql migrate
```

For Prisma or Drizzle Kit, install your dependencies first and run your own tool; the check sets `DATABASE_URL` to its throwaway database:

```yaml
          migrations-dir: prisma/migrations
          migrate-command: npx prisma migrate deploy # or: npx drizzle-kit migrate
```

## Inputs

| Input | Default | |
| --- | --- | --- |
| `project` | | The DbProof project, `org/slug`. |
| `migrations-dir` | | The migrations folder, relative to the repository root. |
| `migrate-command` | | Your migrate command. The throwaway database is in `DBPROOF_CHECK_*` variables and `DATABASE_URL`. |
| `dbproof-url` | `https://app.dbproof.dev` | DbProof's address. |
| `database` | | An empty Postgres to use instead of starting one. |
| `postgres-image` | `postgres:{major}` | The image the check starts, on production's major version, e.g. `pgvector/pgvector:pg{major}`. |
| `fail-on-unreachable` | `false` | Fail rather than pass with a warning when DbProof can't be reached. |

The check runs in your runner and never needs production credentials. The agent behind it is open source: [dbproof/agent](https://github.com/dbproof/agent). Production's snapshot comes from [DbProof capture](https://github.com/dbproof/capture-action).

## License

[Apache-2.0](LICENSE)
