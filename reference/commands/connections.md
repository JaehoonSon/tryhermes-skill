# `hermes connections`

Manage the customer data source that Hermes uses to detect users.

A connection is required before triggers can be created — every `detection_query` runs against a selected connection's source, and the caller writes that query by reading that connection's `schema_snapshot` first.

> **Read [data-sources.md](../data-sources.md) before anything else here.** It defines the supported source types (`postgres`, `mysql`, `csv`, `firestore`, `posthog`), what query language each speaks, the shape of the schema snapshot per source, dialect caveats, and example detection queries. This page covers the CLI surface; data-sources.md covers the model.

## Fields

A connection maps 1:1 to a row in the `connections` table.

| Column                  | Type                | Default           | Settable on `create`                                | Settable on `update` (flag)   | Applies to        | Notes                                                                                                                                                             |
| ----------------------- | ------------------- | ----------------- | --------------------------------------------------- | ----------------------------- | ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                    | uuid                | auto              | ✗                                                   | ✗                             | all               | Primary key.                                                                                                                                                      |
| `org_id`                | uuid                | session           | ✗                                                   | ✗                             | all               | Resolved from the linked org / `--org`.                                                                                                                           |
| `label`                 | text \| null        | source label      | `--label <label>`                                   | planned                       | all               | Human label shown in Settings and `connections list`.                                                                                                             |
| `is_default`            | boolean             | first source only | `--default`                                         | planned                       | all               | Default source used when a trigger create call does not pass a source id.                                                                                         |
| `source_type`           | text                | `"postgres"`      | derived from the `--*-url`/`--*-file` flag on `add` | ✗                             | all               | One of `postgres`, `mysql`, `csv`, `firestore`, `posthog`. Immutable after create.                                                                                |
| `postgres_url`          | text \| null        | `null`            | `--postgres-url <url>`                              | `--postgres-url <url>`        | `postgres`, `csv` | Postgres connection string. For `csv` connections this points at the managed Postgres Hermes loaded the file into. **Encrypted at rest**; redacted in CLI output. |
| `mysql_url`             | text \| null        | `null`            | `--mysql-url <url>`                                 | `--mysql-url <url>`           | `mysql`           | MySQL/MariaDB connection string. **Encrypted at rest**; redacted in CLI output.                                                                                   |
| `firestore_credentials` | text \| null        | `null`            | `--firestore-credentials <json>`                    | same                          | `firestore`       | Firestore service-account JSON. **Encrypted at rest**; redacted in CLI output.                                                                                    |
| `firestore_project_id`  | text \| null        | `null`            | `--firestore-project-id <id>`                       | same                          | `firestore`       |                                                                                                                                                                   |
| `posthog_host`          | text \| null        | `null`            | `--posthog-host <url>`                              | planned                       | `posthog`         | PostHog instance host, e.g. `https://us.posthog.com`.                                                                                                             |
| `posthog_project_id`    | text \| null        | `null`            | `--posthog-project-id <id>`                         | planned                       | `posthog`         | PostHog project id used in `/api/projects/{project_id}/query`.                                                                                                    |
| `posthog_api_key`       | text \| null        | `null`            | `--posthog-api-key <key>`                           | planned                       | `posthog`         | PostHog personal API key. **Encrypted at rest**; redacted in CLI output.                                                                                          |
| `schema_snapshot`       | jsonb \| null       | `null`            | ✗                                                   | refreshed by `inspect-schema` | all               | Introspected schema. Shape depends on `source_type` — see [data-sources.md](../data-sources.md). System-set.                                                      |
| `status`                | text                | `"pending"`       | ✗                                                   | ✗                             | all               | Lifecycle: `pending` → `connected` → `synced` → `failed`. System-set.                                                                                             |
| `last_synced_at`        | timestamptz \| null | `null`            | ✗                                                   | refreshed by `inspect-schema` | all               | Last successful introspection. System-set.                                                                                                                        |
| `suggestions_cache`     | jsonb \| null       | `null`            | ✗                                                   | ✗                             | all               | Cached trigger-intent suggestions. System-set.                                                                                                                    |
| `suggestions_cached_at` | timestamptz \| null | `null`            | ✗                                                   | ✗                             | all               | System-set.                                                                                                                                                       |
| `created_at`            | timestamptz         | now               | ✗                                                   | ✗                             | all               |                                                                                                                                                                   |
| `updated_at`            | timestamptz         | now               | ✗                                                   | ✗                             | all               |                                                                                                                                                                   |

Every JSON response includes every column above. Fields not applicable to the current `source_type` are returned as `null`.

### Canonical JSON shape (Postgres example)

```json
{
  "id": "cn_...-uuid",
  "org_id": "org_...-uuid",
  "label": "Production PostgreSQL",
  "is_default": true,
  "source_type": "postgres",
  "postgres_url": "<redacted>",
  "mysql_url": null,
  "firestore_credentials": null,
  "firestore_project_id": null,
  "posthog_host": null,
  "posthog_project_id": null,
  "posthog_api_key": null,
  "schema_snapshot": {
    "source_type": "postgres",
    "query_language": "sql",
    "example_query": "SELECT id, email FROM users LIMIT 5",
    "tables": [
      {
        "name": "users",
        "row_count": 12453,
        "columns": [{ "name": "id", "type": "uuid" }, "..."]
      }
    ]
  },
  "status": "synced",
  "last_synced_at": "2026-05-13T20:11:04.123Z",
  "suggestions_cache": ["welcome new signups", "..."],
  "suggestions_cached_at": "2026-05-13T20:12:00.000Z",
  "created_at": "2026-05-13T20:10:00.000Z",
  "updated_at": "2026-05-13T20:10:00.000Z"
}
```

For Firestore the `schema_snapshot` instead has `query_language: "firestore"` and a `collections` array with `inferred_fields` and `presence_rate` per field. For PostHog it has `query_language: "hogql"` and a sampled list of common events — see [data-sources.md](../data-sources.md).

## Selecting a source (multi-source orgs)

`show`, `schema`, `query`, `inspect-schema`, and `remove` all act on **one** connection. Three ways to choose it, in precedence order:

1. `--source <id>` (alias `--connection <id>`) — explicit, and the form to prefer in scripts/agents. Wins over everything.
2. The positional `[id]` argument — back-compat; equivalent to `--source` when no flag is given.
3. **Default** — if you name no source and the org has more than one, the CLI uses the org default (`is_default = true`) and prints a one-line hint to stderr naming the chosen source and how to target another. `--json` output is unaffected (the hint goes to stderr only).

For trigger creation, the matching selector is `hermes triggers create --source <id>` → `source_connection_id`.

## Subcommands

### `hermes connections list`

Returns an array of full connection rows.

### `hermes connections add --<source-flag> <value>`

Tells Hermes about another data source. Existing sources and triggers are left intact.

| Flag                                                                     | Resulting `source_type` | Effect when re-run                                                      |
| ------------------------------------------------------------------------ | ----------------------- | ----------------------------------------------------------------------- |
| `--postgres-url <url>`                                                   | `postgres`              | Creates another Postgres source.                                        |
| `--mysql-url <url>`                                                      | `mysql`                 | Creates another MySQL source.                                           |
| `--firestore-credentials <json-or-@file> [--firestore-project-id <id>]`  | `firestore`             | Creates another Firestore source.                                       |
| `--posthog-host <url> --posthog-project-id <id> --posthog-api-key <key>` | `posthog`               | Creates another PostHog source.                                         |
| `--csv-file <path> [--table-name <name>]`                                | `csv`                   | Creates another CSV source and ingests the file as a table.             |
| `--connection <id>`                                                      | `csv`                   | With `--csv-file`, appends the CSV as another table in that CSV source. |
| `--label <label>`                                                        | all                     | Sets a human-readable label.                                            |
| `--default`                                                              | all                     | Makes the new source the org default for trigger creation fallbacks.    |

The first source in an org becomes default automatically. Later sources are non-default unless `--default` is provided.

Examples:

```
hermes connections add --postgres-url 'postgres://readonly:...@host/db'
hermes connections add --mysql-url    'mysql://readonly:...@host:3306/db'
hermes connections add --csv-file     './users.csv' --label 'Conference leads'
hermes connections add --csv-file     './orders.csv' --connection <csv-id> --table-name customer_orders
hermes connections add --firestore-credentials @sa.json --firestore-project-id my-project
hermes connections add --posthog-host https://us.posthog.com --posthog-project-id 12345 --posthog-api-key phx_...
```

For all sources, Hermes validates the credential / file before saving, introspects the schema, and persists the connection in one synchronous call. Returns the full connection row.

`--confirm-switch` is deprecated. `connections add` no longer replaces an existing source.

**Recommendation:** for SQL sources, use a **read-only** user. Hermes only needs `SELECT` on user-related tables. For Firestore, scope the service account to read-only. For PostHog, use the least-privileged personal API key that can run query reads for the project.

### `hermes connections show <id>`

Returns the full connection row (see Canonical JSON shape above). Secret columns are returned as `"<redacted>"`.

### `hermes connections schema [<id>]`

Convenience wrapper that returns just the `schema_snapshot` portion of the connection row, formatted for reading. Always leads with `source_type`, `query_language`, and `example_query` so the caller knows which dialect to write in. This is the **first command the caller should run before writing any `detection_query`**.

```
hermes connections schema
```

If the org has multiple connections, omit the id to use the default source (a stderr hint names it), or target a specific one with `--source <id>` (or the positional id).

### `hermes connections query "<q>" [<id>]`

Runs the query read-only against the connection's source, returns rows + count + sample. **Does not create a trigger or send anything.** Use this to iterate on a candidate `detection_query` before calling `hermes triggers create`.

The `<q>` payload shape depends on `source_type`:

- **SQL family** (`postgres`, `mysql`, `csv`): a SQL string in the right dialect.
- **Firestore**: a JSON DSL object — `{ collection, where?, orderBy?, limit? }`. Pass it as a JSON string on the CLI.
- **PostHog**: a HogQL string.

```
hermes connections query "SELECT id, email FROM users WHERE created_at > now() - interval '7 days' LIMIT 50"
hermes connections query '{"collection":"users","where":[["tier","==","pro"]],"limit":50}'
hermes connections query "SELECT distinct_id AS id, properties.email AS email FROM events LIMIT 50"
```

If you send the wrong dialect, the error response identifies the actual `source_type` and includes the example query for that source.

### `hermes connections inspect-schema [<id>]`

Re-runs introspection synchronously against the live source. Updates `schema_snapshot` and `last_synced_at` and returns the refreshed connection row. Use after schema changes on the customer source. Accepts `--source <id>` (or the positional id); defaults to the org default.

### `hermes connections remove [<id>] --confirm [--csv-table <name>]`

Two operations on the same verb, separated by whether `--csv-table` is present.

**Whole-connection remove** (`remove --confirm`):
Detaches the entire connection. For CSV connections, this also drops every loaded table from the managed Postgres and removes the original CSV files from storage. Hermes blocks removal while triggers still depend on the connection, and it blocks removing the default connection while other sources exist.

**Per-table remove** (`remove --csv-table <name> --confirm`):
For `csv` connections only. Drops the named table from managed Postgres, removes its CSV file from storage, deletes the dataset row, and refreshes the connection's `schema_snapshot`. Other tables in the same CSV connection are unaffected. Errors with `INVALID_OPERATION` if the connection isn't `csv`.

```
hermes connections remove --confirm                        # remove the whole connection
hermes connections remove --csv-table customer_orders --confirm   # drop just one CSV table
```

## Notes for agents

- **The schema is the contract.** Always run `hermes connections schema` before writing a detection query. The response's `query_language` field tells you which dialect to use; the `example_query` field is a literal starting point.
- **Test queries before committing to a trigger.** `hermes connections query` is read-only and free of side effects — iterate there. Once the row count + sample look right, pass the exact same query string to `hermes triggers create --detection-query`.
- **Don't infer the dialect from the URL or label.** A `csv` connection looks like a CSV but speaks Postgres SQL. A `mysql` URL looks like Postgres but rejects `interval '7 days'`. PostHog uses HogQL, not the SQL-family toolchain. Trust `source_type` and `query_language` from the schema response, nothing else.

## Common errors

| Code                          | Meaning                                                    | Hint                                                                                                                                                          |
| ----------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `CONNECTION_FAILED`           | Cannot reach the customer source; `status` set to `failed` | Verify credentials and that the host accepts external connections                                                                                             |
| `SCHEMA_INTROSPECTION_FAILED` | Connected but cannot read schema                           | For SQL: verify the user has `SELECT` on `pg_catalog` / `information_schema`. For Firestore: verify the service account has `datastore.viewer` or equivalent. |
| `NOT_FOUND`                   | A requested connection id does not exist in this org       | Run `hermes connections list` and retry with the correct id                                                                                                   |
| `INVALID_OPERATION`           | `--csv-table` passed to a non-CSV connection               | Per-table remove is only valid for `source_type=csv`                                                                                                          |
| `QUERY_DIALECT_MISMATCH`      | Sent SQL to a Firestore connection (or vice versa)         | Re-read `hermes connections schema` for the correct dialect + example                                                                                         |
| `QUERY_FAILED`                | Query reached the source but errored                       | Read the underlying message; common causes are typos, missing columns, missing Firestore composite indexes                                                    |
