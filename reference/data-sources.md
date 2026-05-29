# Data sources

Hermes is data-source-agnostic by design. A trigger's `detection_query` is whatever query language the connected source speaks — not always SQL. The canonical agent flow is:

```
hermes auth login                      # 1. authenticate
hermes connections add ...             # 2. connect a data source (picks source_type)
hermes connections schema              # 3. read the schema — see source_type, dialect, tables/collections
hermes connections query "<q>"         # 4. iterate on the detection query until it returns the right cohort
hermes triggers create --detection-query "<q>" --email-prompt "..." ...   # 5. submit the trigger
```

**Step 3 is load-bearing.** The schema response leads with `source_type` and `query_language`, plus an example query in the right dialect. Always read it before writing a query — pattern-matching on SQL when the source is Firestore wastes a round trip.

## Supported source types

| `source_type` | Query language              | Family    | Notes                                                                                                                                                                                                                      |
| ------------- | --------------------------- | --------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `postgres`    | SQL (Postgres dialect)      | SQL       | Primary, fully-featured.                                                                                                                                                                                                   |
| `mysql`       | SQL (MySQL/MariaDB dialect) | SQL       | Same tools as Postgres, **dialect differences matter** — see below.                                                                                                                                                        |
| `csv`         | SQL (Postgres dialect)      | SQL       | The user uploads a CSV; Hermes loads it into a managed Postgres and treats it like one. From the agent's perspective, identical to `postgres`. The `csv` label exists only so the UI/user knows where the data originated. |
| `firestore`   | Firestore JSON DSL          | Document  | NoSQL. No joins, no aggregations, restricted query shape. Reads are metered.                                                                                                                                               |
| `posthog`     | HogQL                       | Analytics | Product analytics events/persons queried through PostHog's Query API. SQL-like, but not Postgres/MySQL SQL. No cross-source joins.                                                                                         |

The `query_language` field on the schema response is what the agent actually dispatches on. Today:

- `sql` → write SQL. Dialect is Postgres unless `source_type === "mysql"`.
- `firestore` → write a JSON query object (DSL described below).
- `hogql` → write HogQL for PostHog events/persons.

If a future source family lands (BigQuery, MongoDB, etc.), it will appear here with its own `query_language` value. Read the schema; don't hard-code source types.

---

## SQL family: `postgres`, `mysql`, `csv`

### Schema snapshot shape

```json
{
  "source_type": "postgres",
  "query_language": "sql",
  "example_query": "SELECT id, email FROM users LIMIT 5",
  "tables": [
    {
      "name": "users",
      "row_count": 12453,
      "columns": [
        { "name": "id", "type": "uuid" },
        { "name": "email", "type": "text" },
        { "name": "created_at", "type": "timestamptz" },
        { "name": "subscription_tier", "type": "text" }
      ]
    }
  ]
}
```

### Example detection query

```sql
SELECT u.id, u.email
FROM users u
LEFT JOIN onboarding_steps os ON u.id = os.user_id AND os.step = 'complete'
WHERE os.id IS NULL
  AND u.created_at > now() - interval '14 days'
```

Required projection: at minimum `id` and `email`. The pipeline scans the result set per row; each row → one job.

### Postgres dialect (default)

Standard PostgreSQL. `ILIKE`, `RETURNING`, `jsonb_*`, `generate_series`, array types, `DISTINCT ON`, `interval '7 days'`, `to_char`, `date_trunc` — all available.

### MySQL dialect (`source_type: "mysql"`)

Same SQL family, but several Postgres idioms don't work. When `source_type === "mysql"`:

- **Quote identifiers with backticks**, not double-quotes. `` SELECT `first name` FROM `users` `` — not `"first name"`.
- **String literals use single quotes** only. Don't rely on double quotes; behavior depends on `sql_mode`.
- **No `ILIKE`.** Use `LIKE` with a case-insensitive collation (most setups default to `_ci`), or wrap both sides in `LOWER(...)`.
- **No `RETURNING`** on writes (irrelevant here — `connections query` is read-only — but don't propose it).
- **`LIMIT` with offset** is `LIMIT offset, count` or `LIMIT count OFFSET offset`.
- **Dates:** use `DATE_FORMAT(col, '%Y-%m-%d')`, `NOW()`, `CURDATE()`, `DATE_SUB(NOW(), INTERVAL 7 DAY)`. Postgres' `to_char`, `date_trunc`, and `INTERVAL '7 days'` (quoted) don't work.
- **JSON:** use `JSON_EXTRACT(col, '$.path')` or the `->` / `->>` operators. Not `jsonb_*`.
- **Boolean:** stored as `TINYINT(1)`. `true`/`false` literals work but results come back as `0`/`1`.
- **No `generate_series`, no array types, no `DISTINCT ON`.** For "first row per group," use a window function (MySQL 8+): `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)`.
- **Schema lookups** scope via `information_schema.columns WHERE TABLE_SCHEMA = DATABASE()`. There is no `public` schema.

### CSV (`source_type: "csv"`)

Each uploaded CSV becomes its own **table** inside a CSV source. Hermes ingests the file into a managed Postgres instance — for query purposes the source behaves exactly like `postgres` and you write Postgres-flavored SQL.

Adding more data is multi-step in a way the other source types aren't:

```
hermes connections add --csv-file ./users.csv                         # creates a CSV source + ingests users table
hermes connections add --csv-file ./orders.csv --table-name orders    # creates another CSV source
hermes connections add --csv-file ./events.csv --connection <csv-id>  # appends another table to that CSV source
```

Re-running `add --csv-file` with `--connection <csv-id>` **accumulates** tables inside that CSV source. Without `--connection`, it creates a separate CSV source. Other source types also create separate sources. Per-table removal is `hermes connections remove --csv-table <name> --confirm` — see [connections.md](commands/connections.md#hermes-connections-remove-id---confirm---csv-table-name).

The `schema_snapshot` is the union of every ingested table's columns. Cross-table joins work because everything lives in the same managed Postgres.

---

## Document family: `firestore`

### Schema snapshot shape

```json
{
  "source_type": "firestore",
  "query_language": "firestore",
  "example_query": {
    "collection": "users",
    "where": [["created_at", ">", "2026-05-01"]],
    "limit": 5
  },
  "collections": [
    {
      "name": "users",
      "sampled_doc_count": 500,
      "inferred_fields": [
        { "name": "email", "type": "string", "presence_rate": 1.0 },
        { "name": "tier", "type": "string", "presence_rate": 0.92 },
        { "name": "lifetimeValue", "type": "number", "presence_rate": 0.61 },
        { "name": "lastSeenAt", "type": "timestamp", "presence_rate": 0.88 }
      ]
    }
  ]
}
```

### Critical caveats (read before writing any Firestore query)

- **Schema is inferred, not declared.** Each field has a `presence_rate`. A 90%+ field is reliable; a 20% field is brittle. Fields below 5% may not have been sampled at all. Treat schema as a strong hint, not ground truth.
- **No joins.** You cannot correlate across collections in one query. If you need users _and_ their orders, fetch users first, then orders by ID per user. This is expensive — be deliberate.
- **No aggregations.** No SUM, COUNT, AVG, GROUP BY. If you need "top spenders," look for a denormalized field on the user doc (e.g. `lifetimeValue`, `totalOrders`) — most Firebase apps maintain these. If the field doesn't exist, the cohort may not be computable.
- **Limited query shape per collection:**
  - At most one inequality field per query (`<`, `<=`, `>`, `>=`, `!=`).
  - If you use an inequality, the first `orderBy` field must match it.
  - No substring / `LIKE` / regex.
  - `array-contains` matches one value inside an array field; `array-contains-any` takes up to 10 values.
- **Composite indexes** can be required for multi-field queries. The error will say so. Hermes cannot create indexes — surface to the user.
- **Reads cost money.** Every document returned counts against a per-run budget. Use `limit` aggressively while exploring (10–25 is plenty for hypothesis testing).

### DSL shape

```json
{
  "collection": "users",
  "where": [
    ["tier", "in", ["pro", "enterprise"]],
    ["lastSeenAt", ">", "2026-05-01T00:00:00Z"]
  ],
  "orderBy": [{ "field": "lastSeenAt", "direction": "desc" }],
  "limit": 100
}
```

- `collection` (string, required) — top-level collection name.
- `where` (array of triples, optional) — `[field, op, value]`. Ops: `==`, `!=`, `<`, `<=`, `>`, `>=`, `in`, `not-in`, `array-contains`, `array-contains-any`.
- `orderBy` (array, optional) — must include the inequality field as the first sort key if any inequality is used.
- `limit` (number, optional) — cap on documents returned. Server enforces a max.

### Example detection query (Firestore)

```json
{
  "collection": "users",
  "where": [
    ["tier", "==", "pro"],
    ["onboardingCompleted", "==", false]
  ],
  "limit": 200
}
```

Required: each returned document must have an `email` field (or a denormalized `email` projected via field path). The pipeline reads `email` (and `id` from the document path) to enqueue per-recipient jobs.

---

## Analytics family: `posthog`

### Schema snapshot shape

```json
{
  "source_type": "posthog",
  "query_language": "hogql",
  "example_query": "SELECT distinct_id AS id, properties.email AS email FROM events LIMIT 5",
  "events": {
    "event": "String event name",
    "distinct_id": "String actor identifier",
    "timestamp": "DateTime event timestamp",
    "properties": "JSON event properties, address with properties.<key>"
  },
  "common_events_last_30_days": {
    "checkout_completed": "1,204 events in last 30 days"
  }
}
```

### Critical caveats

- **HogQL is not Postgres SQL.** It uses PostHog's analytics query surface. Use `query_language: "hogql"` and start from the example query.
- **Events are not a canonical users table.** Recipient identity usually comes from `distinct_id`, `person_id`, or dynamic properties like `properties.email`. Verify the property exists with a small query before using it in a trigger.
- **No cross-source joins.** A PostHog query cannot join directly to a Postgres/MySQL/CSV/Firestore connection. Use a shared identifier in PostHog, or defer cross-source reasoning to a later analysis/export layer.
- **Triggers still need recipients.** Detection queries must return `id` and `email` when rows exist, just like SQL-family sources.

### Example detection query (PostHog)

```sql
SELECT
  distinct_id AS id,
  properties.email AS email,
  event,
  timestamp
FROM events
WHERE event = 'checkout_completed'
  AND timestamp >= now() - INTERVAL 7 DAY
LIMIT 200
```

---

## How the agent decides which dialect to write

The agent flow is identical regardless of source:

1. **Always start with the schema.** `hermes connections schema` returns the source type, query language, an example query, and the structural snapshot. This is the contract.
2. **Write a candidate query in the matching dialect.** SQL for SQL family, JSON DSL for Firestore, HogQL for PostHog. The example_query field is a literal template — copy and adapt.
3. **Test before submitting.** `hermes connections query "<q>"` runs the query read-only against the customer's source, returns a sample + count. Iterate here, not on the trigger row.
4. **Submit the validated query.** `hermes triggers create --detection-query "<q>" --email-prompt "..."` persists the trigger.

If the caller sends the wrong dialect (SQL to a Firestore connection, JSON to a string-query source, etc.), the error response identifies the actual `source_type` and includes the example query for that source — recoverable in one round trip.

## What stays the same across sources

- **Email prompt** is plain text in every case. The drafter is source-agnostic.
- **Operational fields** (cooldown, recurrence, scan interval, sender, auto-send, tracking, footer/address) are independent of `source_type`.
- **The pipeline** (scan → evaluate → draft → review → deliver) doesn't care. Source-specific logic ends at the detection step.
- **The required projection** is conceptually the same: each result must yield enough to identify and email a person. SQL/HogQL projects `id, email`; Firestore returns a document whose path supplies the id and whose body supplies the email.
