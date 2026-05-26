# Create a trigger

A trigger is a campaign config: a **detection query** (who to find) plus an **email prompt** (what to say). You — the caller, agent or human — write both. Hermes validates against the live source, persists the trigger, and runs the pipeline.

There is **no chat agent** in this flow. Hermes is the platform; the reasoning loop is yours.

## The flow

```
hermes connections schema                                 # 1. learn source_type + dialect
hermes connections query "<candidate>"                    # 2. iterate the query until the cohort is right
hermes triggers create \
  --name "..." \
  --detection-query "<the validated query>" \
  --email-prompt "..." \
  [--sender <id>] [...operational flags]                  # 3. submit (created active)
```

The trigger is **created active** — it starts firing on its schedule immediately, with the first run's drafts held for approval (auto-send is off by default). `create` returns a `detection_preview` so you see the live audience size without a separate call. Pause with `hermes triggers pause <id>` if you want to hold it.

## Step 1 — Read the schema

```
hermes connections schema
```

The response leads with `source_type`, `query_language`, and an `example_query`. **Always read this first.** If the source is Firestore, the example is a JSON object; if SQL, it's a SQL string. The full schema (tables/columns or collections/fields) is in the same payload.

See [data-sources.md](../reference/data-sources.md) for the canonical per-source matrix — dialect caveats, example detection queries, and the Firestore DSL shape.

## Step 2 — Write and test the detection query

Project the columns/fields the pipeline needs: at minimum `id` and `email`.

```
# SQL (Postgres / MySQL / CSV)
hermes connections query "SELECT id, email FROM users WHERE created_at > now() - interval '24 hours' LIMIT 50"

# Firestore
hermes connections query '{"collection":"users","where":[["tier","==","pro"]],"limit":50}'
```

The response includes a count and a sample. **Iterate here, not on the trigger row** — `hermes connections query` is read-only and free of side effects.

Check before you submit:

- Does the row/document count match your expectation? (10 when you expected 1,000, or 50,000 when you expected 100, is a red flag.)
- Does each result include `email` (and an identifier)?
- For Firestore: does your query stay inside the [restrictions](../reference/data-sources.md#critical-caveats-read-before-writing-any-firestore-query) (one inequality, no joins, no aggregations)?

## Step 3 — Submit the trigger

Once the query looks right, pass the exact same string to `triggers create`:

```
hermes triggers create \
  --name "Welcome 24h signups" \
  --detection-query "SELECT id, email FROM users WHERE created_at > now() - interval '24 hours'" \
  --email-prompt "Write a warm, short welcome email. Mention the signup date naturally; keep it under 4 sentences." \
  --sender snd_... \
  --cooldown-hours 168
```

Hermes re-runs the query against the source as part of validation. If the query fails or the projection is missing required fields, the response includes the underlying error and the example query for the connection's dialect.

## Step 4 — (Optional) preview

```
hermes triggers preview <trigger-id>
```

Dry-run the detection query: returns the count of users that would match **right now** plus a sample. Does not draft or send anything. Useful to confirm the audience size hasn't shifted since you created the trigger.

## Step 4 — Make sure it can send

The trigger is already active, but sends fail without a sender on a verified domain. Set one if you didn't pass `--sender` on create:

```
hermes triggers update <id> --sender <sender-id>
```

The errors are explicit (`NO_SENDER`, `DOMAIN_NOT_VERIFIED`) with hints pointing at the next command. Pause the trigger any time with `hermes triggers pause <id>`.

## Writing a good detection query

Two qualities to optimize for:

**1. The right people.** Quantify "power user" / "churned" / "engaged." Vague intents become brittle queries.

| Vague intent | Better translation |
|---|---|
| "email power users" | `tier IN ('pro','enterprise') AND last_login_at > now() - interval '30 days'` |
| "win back churned users" | `last_login_at < now() - interval '30 days' AND last_login_at > now() - interval '180 days'` |
| "thank loyal customers" | `lifetime_value > 500` |

**2. The right shape.** The pipeline reads `id` and `email` from each result. If the column is named differently, alias it: `SELECT user_id AS id, contact_email AS email FROM ...`. For Firestore, return documents whose body includes `email`.

## Writing a good email prompt

The prompt is a **brief**, not a template. It tells the drafter what to write, not what to literally output. Good prompts include:

- **Audience context** ("the recipient just signed up", "the recipient is a pro-tier user who hasn't logged in for 14 days")
- **Tone and length** ("warm, conversational, 3–4 sentences max")
- **What to mention naturally** ("their signup date if relevant", "the feature they last used")
- **What to avoid** ("don't use marketing-speak", "don't ask them to upgrade")

The drafter has access to the row Hermes scanned, prior thread history, and org identity/memory — you don't need to enumerate facts; the drafter can pull them.

## Iterating

Everything is editable via `hermes triggers update <id>` — operational fields (`--sender`, `--cooldown-hours`, `--auto-send`, …) and the content fields too:

```
hermes triggers update <id> --detection-query "<new query>"   # re-validated against the live source
hermes triggers update <id> --email-prompt "<new brief>"
```

Editing `--detection-query` re-runs the full validation pipeline (dialect match + dry-run + `id`/`email` projection). A bad query is rejected and the trigger is left unchanged, so test with `hermes connections query` first. If you're pivoting to a fundamentally different audience it's often cleaner to delete and recreate, but for tweaks, `update` is in place and safe.

## After activation

The pipeline takes over. Use:

- `hermes triggers run <id>` — fire now, in addition to the schedule
- `hermes drafts list --trigger <id>` — see what got drafted
- `hermes analytics campaign <id> --since 24h` — see how it's performing

See [review-drafts.md](review-drafts.md) for the approve/reject workflow.
