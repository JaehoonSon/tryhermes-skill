# `hermes triggers`

Manage campaign triggers. A trigger pairs an audience (who) with an intent (what to say). The caller (you — agent or human) writes the detection query in the connected source's native dialect and provides an email prompt; Hermes validates the query against the live source, persists the trigger, and runs the pipeline.

> The `detection_query` is **not always SQL.** It's whatever query language the connection speaks — Postgres SQL, MySQL SQL, Firestore JSON DSL, or PostHog HogQL. Read [data-sources.md](../data-sources.md) and the connection's `schema_snapshot` first so you write in the right dialect. Iterate with `hermes connections query` until the result looks right, then submit the same query string to `hermes triggers create`.

## Fields

A trigger maps 1:1 to a row in the `triggers` table. All fields are returned by every `show`, `list`, `create`, and `update` response.

| Column                     | Type            | Default           | Settable on `create`                    | Settable on `update` (flag)             | Notes                                                                                                                                                                                                                                                                  |
| -------------------------- | --------------- | ----------------- | --------------------------------------- | --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`                       | uuid            | auto              | ✗                                       | ✗                                       | Primary key. Use this everywhere the docs say `<id>`.                                                                                                                                                                                                                  |
| `org_id`                   | uuid            | session           | ✗                                       | ✗                                       | Resolved from the linked org / `--org`.                                                                                                                                                                                                                                |
| `name`                     | text            | auto-generated    | `--name`                                | `--name`                                | Human label. Auto-generated from intent if omitted.                                                                                                                                                                                                                    |
| `type`                     | text            | `"custom"`        | `--type`                                | `--type`                                | Free-form category tag (e.g. `"custom"`, `"onboarding"`, `"reactivation"`).                                                                                                                                                                                            |
| `detection_query`          | text            | —                 | `--detection-query <q>`                 | `--detection-query <q>`                 | The query that selects recipient rows. Source-native dialect — SQL for `postgres`/`mysql`/`csv`, JSON DSL for `firestore`, HogQL for `posthog`. Caller-written; see [data-sources.md](../data-sources.md). Hermes validates against the live source before persisting. |
| `email_prompt`             | text            | —                 | `--email-prompt <text>`                 | `--email-prompt <text>`                 | The prompt that drives per-user drafting. Plain text, source-agnostic.                                                                                                                                                                                                 |
| `sender_identity_id`       | uuid \| null    | `null`            | `--sender`                              | `--sender`                              | FK → `email_sender_identities.id`. Required before `activate`.                                                                                                                                                                                                         |
| `auto_send`                | boolean         | `false`           | `--auto-send` / `--no-auto-send`        | `--auto-send` / `--no-auto-send`        | If true, skip drafts and send on detection.                                                                                                                                                                                                                            |
| `cooldown_hours`           | integer         | `72`              | `--cooldown-hours <n>`                  | `--cooldown-hours <n>`                  | Don't re-fire for the same user within this window.                                                                                                                                                                                                                    |
| `is_recurring`             | boolean         | `false`           | `--recurring` / `--no-recurring`        | `--recurring` / `--no-recurring`        | Allow re-fire on the same user after `recurrence_cooldown_days`.                                                                                                                                                                                                       |
| `recurrence_cooldown_days` | integer \| null | `null`            | `--recurrence-cooldown-days <n>`        | `--recurrence-cooldown-days <n>`        | Days before a user becomes eligible again (recurring only).                                                                                                                                                                                                            |
| `scan_interval_minutes`    | integer         | `60`              | `--scan-interval-minutes <n>`           | `--scan-interval-minutes <n>`           | How often the scheduler fires the trigger.                                                                                                                                                                                                                             |
| `category`                 | text            | `"transactional"` | `--category <transactional\|marketing>` | `--category <transactional\|marketing>` | Affects compliance + unsubscribe behavior.                                                                                                                                                                                                                             |
| `track_opens`              | boolean         | `true`            | `--track-opens` / `--no-track-opens`    | same                                    | Insert open-tracking pixel.                                                                                                                                                                                                                                            |
| `track_clicks`             | boolean         | `true`            | `--track-clicks` / `--no-track-clicks`  | same                                    | Rewrite links for click tracking.                                                                                                                                                                                                                                      |
| `is_active`                | boolean         | `true`            | created active                          | via `activate` / `pause`                | New triggers are created **active**. Toggle later with `hermes triggers activate`/`pause`, not `update`.                                                                                                                                                               |
| `source_session_id`        | text \| null    | `null`            | ✗                                       | ✗                                       | Chat-session origin, if created from a session (web). Always `null` for CLI-created triggers. System-set.                                                                                                                                                              |
| `source_connection_id`     | uuid \| null    | inferred default  | `--source <id>`                         | `--source <id>`                         | Connection used for detection. If omitted, Hermes uses the org default source.                                                                                                                                                                                         |
| `source_context`           | jsonb \| null   | `null`            | ✗                                       | ✗                                       | Mentions metadata captured at creation (tables/columns referenced). System-set.                                                                                                                                                                                        |
| `created_at`               | timestamptz     | now               | ✗                                       | ✗                                       |                                                                                                                                                                                                                                                                        |
| `updated_at`               | timestamptz     | now               | ✗                                       | ✗                                       |                                                                                                                                                                                                                                                                        |

Every JSON response includes every column above.

### Canonical JSON shape

```json
{
  "id": "tr_abc123-...-uuid",
  "org_id": "org_...-uuid",
  "name": "Welcome 24h signups",
  "type": "onboarding",
  "detection_query": "SELECT id, email, name FROM users WHERE created_at > now() - interval '24 hours'",
  "email_prompt": "Write a warm, short welcome email...",
  "sender_identity_id": "snd_...-uuid",
  "auto_send": false,
  "cooldown_hours": 72,
  "is_recurring": false,
  "recurrence_cooldown_days": null,
  "scan_interval_minutes": 60,
  "category": "transactional",
  "track_opens": true,
  "track_clicks": true,
  "is_active": true,
  "source_session_id": null,
  "source_connection_id": "cn_...-uuid",
  "created_at": "2026-05-13T20:11:04.123Z",
  "updated_at": "2026-05-13T20:11:04.123Z"
}
```

> `create` returns this row **plus** a `detection_preview` object — `{ "count": <n>, "sample": [...] }` — the live dry-run Hermes ran while validating, so you see the current audience size without a separate `preview` call.

## Authoring a trigger

The recommended flow (caller-driven, no chat agent):

```
hermes connections schema                                 # 1. learn source_type + dialect
hermes connections query "<candidate detection query>"    # 2. iterate until the cohort is right
hermes triggers create \
  --name "..." \
  --detection-query "<the validated query>" \
  --email-prompt "<the brief for per-user copy>" \
  [--sender <id>] [...other operational flags]            # 3. submit
```

When the org has multiple sources, pass the same connection id to `connections schema`, `connections query`, and `triggers create --source <id>`. If `--source` is omitted, Hermes validates and stores the trigger against the org default source.

You — the caller — are the reasoning loop. Hermes is the platform: it validates the query you submit against the live source, persists the trigger, and runs the pipeline.

### `hermes triggers create --detection-query <q> --email-prompt <text> [flags]`

Validates and persists a trigger. Validation runs server-side:

1. **Dialect match** — the query is a SQL string for `postgres`/`mysql`/`csv`, a HogQL string for `posthog`, or a Firestore JSON DSL object for `firestore`. A mismatch returns `QUERY_DIALECT_MISMATCH`.
2. **Dry-run** — Hermes runs the query read-only against the selected source, confirms it executes without error, and captures a row count + sample. A zero-row count is allowed (the audience may populate later) and returned in `detection_preview` so you can decide whether to proceed.
3. **Required projection** — when the dry-run returns at least one row, it must include `id` and `email`. (A zero-row result skips this check.)
4. **Sender** — if `--sender` is set, the sender must exist in the org. Domain-verified is enforced at send time, not here.

New triggers are created **active** (`is_active = true`) and **not** auto-sending (`auto_send = false`) — the first run's drafts wait for human approval. Returns the full trigger row plus `detection_preview` on success. On validation failure, the response includes the underlying source error.

```
# Postgres / MySQL / CSV
hermes triggers create \
  --name "Welcome 24h signups" \
  --detection-query "SELECT id, email FROM users WHERE created_at > now() - interval '24 hours'" \
  --email-prompt "Write a warm, short welcome email mentioning their signup date naturally."

# Firestore
hermes triggers create \
  --name "Pro tier check-in" \
  --detection-query '{"collection":"users","where":[["tier","==","pro"]],"limit":200}' \
  --email-prompt "Check in with pro-tier users who haven't logged in this week..."

# PostHog
hermes triggers create \
  --name "Recent checkout follow-up" \
  --detection-query "SELECT distinct_id AS id, properties.email AS email FROM events WHERE event = 'checkout_completed' LIMIT 200" \
  --email-prompt "Follow up with users who completed checkout and ask about their setup experience."
```

Useful flags on `create`:

- `--name <name>` — human-readable name (auto-generated from the query if omitted)
- `--type <type>` — free-form category tag (default `"custom"`)
- `--source <id>` — source connection used for detection (default source if omitted)
- All operational flags from the `update` table below (`--sender`, `--cooldown-hours`, `--auto-send`, etc.) — settable at create time too

### `hermes triggers preview <id>`

Dry-run the detection query: returns the count of users that would match **right now** plus a sample. Does **not** create drafts or send anything. Useful for sanity-checking before activating or after the source data has changed.

Returns:

```json
{
  "trigger_id": "tr_...-uuid",
  "count": 87,
  "sample": [{ "id": "u_...", "email": "alice@example.com", "...": "..." }]
}
```

`sample` columns are whatever the detection query projects (capped at 5 rows). For Firestore, sample entries are the matching documents (capped at 5).

## Operational commands (literal flags)

### `hermes triggers list [--status active|paused]`

Returns an array of full trigger rows (same shape as Canonical JSON shape above). `--status active` filters to `is_active = true`; `--status paused` filters to `is_active = false`.

### `hermes triggers show <id>`

Returns the full trigger row, plus a `recent_runs` array of the last 10 entries from `job_runs` for this trigger.

### `hermes triggers update <id> [flags]`

Every flag below writes to the column with the same `snake_case` name (e.g. `--cooldown-hours 24` writes `cooldown_hours = 24`).

| Flag                                    | Column written                                                                                                                                       |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--name <name>`                         | `name`                                                                                                                                               |
| `--detection-query <q>`                 | `detection_query` — **re-validated** against the live source before persisting (same dialect + dry-run + `id`/`email` projection checks as `create`) |
| `--source <id>`                         | `source_connection_id` — re-validates the current or supplied detection query against that source before persisting                                  |
| `--email-prompt <text>`                 | `email_prompt`                                                                                                                                       |
| `--sender <id>`                         | `sender_identity_id`                                                                                                                                 |
| `--auto-send` / `--no-auto-send`        | `auto_send`                                                                                                                                          |
| `--cooldown-hours <n>`                  | `cooldown_hours`                                                                                                                                     |
| `--recurring` / `--no-recurring`        | `is_recurring`                                                                                                                                       |
| `--recurrence-cooldown-days <n>`        | `recurrence_cooldown_days`                                                                                                                           |
| `--scan-interval-minutes <n>`           | `scan_interval_minutes`                                                                                                                              |
| `--category <transactional\|marketing>` | `category`                                                                                                                                           |
| `--track-opens` / `--no-track-opens`    | `track_opens`                                                                                                                                        |
| `--track-clicks` / `--no-track-clicks`  | `track_clicks`                                                                                                                                       |

Returns the full updated trigger row. Editing `--detection-query` runs the full validation pipeline — a bad query is rejected and the trigger is left unchanged. Test with `hermes connections query` first.

### `hermes triggers activate <id>` / `hermes triggers pause <id>`

Sets `is_active = true` / `false`. Returns the full row.

### `hermes triggers run <id>`

Fire the trigger immediately (in addition to its schedule). Returns a `job_id`; CLI polls until detection completes by default.

### `hermes triggers delete <id> --confirm`

Removes the trigger. Historical emails survive — the application detaches the `trigger_id` foreign key on `emails` before deleting the row.

## Notes for agents

- **You are the reasoning loop.** Hermes does not run a chat agent to write your detection query. The flow is: read the schema, write a candidate query, test it with `hermes connections query`, then submit it to `triggers create`. Iterate on the _query_, not on a conversation.
- **The `detection_query` is dialect-specific.** Read the connection's `schema_snapshot.query_language` (or [data-sources.md](../data-sources.md)) before writing. SQL family ≠ Firestore ≠ PostHog; Postgres SQL ≠ MySQL SQL.
- **Default to `auto_send: false`** for any new trigger. Have a human approve drafts at least for the first run; switch to `auto_send` only after you've confirmed quality.
- After `create` returns a trigger id, the typical next steps are: `update --sender <id>` (if you didn't set it on create), then `preview` to sanity-check, then `activate`.
- The `Fields` table is the source of truth. If you're considering writing a column, find it in the table — if it has no flag, it's not settable.

## Common errors

| Code                      | Meaning                                                                   | Hint                                                                                                       |
| ------------------------- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| `NO_CONNECTION`           | Can't create trigger without a connection                                 | Run `hermes connections add` first                                                                         |
| `NO_SENDER`               | Trigger has no `sender_identity_id`; can't activate or send               | Run `hermes triggers update <id> --sender <sender-id>`                                                     |
| `DOMAIN_NOT_VERIFIED`     | Sender's domain isn't verified                                            | Verify the domain first                                                                                    |
| `QUERY_DIALECT_MISMATCH`  | Sent SQL to a Firestore connection (or vice versa)                        | Re-read `hermes connections schema` for the correct dialect + example                                      |
| `DETECTION_QUERY_INVALID` | Query doesn't parse or is missing required projection (`id`, `email`)     | Adjust the query; test with `hermes connections query`                                                     |
| `SCHEMA_OUT_OF_DATE`      | `detection_query` references a column/collection not in the cached schema | Run `hermes connections inspect-schema <id>`                                                               |
| `DETECTION_QUERY_FAILED`  | Query ran but errored on the customer source                              | Read the underlying message; common causes are typos, missing columns, missing Firestore composite indexes |
