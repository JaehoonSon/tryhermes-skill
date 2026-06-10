# Getting started: from auth to first send

Goal: take a fresh Hermes account from zero to one delivered email — entirely from the CLI, no web onboarding required.

The setup chain is **auth → (orgs create) → link → connection → domain → sender → trigger**. Skip a step and the next one errors with a hint pointing back. Walk it once and the rest is iteration.

## Prerequisites

- An account on app.hermes.dev (sign up on the dashboard once — there's no CLI sign-up flow yet).
- Credentials for at least one supported data source — Postgres URL, MySQL URL, Firestore service account, a CSV file, or a PostHog project. You can connect several; see [data-sources.md](../reference/data-sources.md) for the full matrix.
- A domain you control DNS for.

## 1. Authenticate

```
hermes auth login        # opens browser, asks you to approve a code
hermes auth whoami       # (optional) confirm the session
```

The CLI starts a device-code flow, prints a URL + short code, and waits while you approve in your browser. The resulting session token is account-level — it can act on any org you're a member of.

For CI / scripted automation, mint an org-scoped API key in the dashboard and pass it directly:

```
hermes auth login --key hk_live_...
```

## 2. Create or select an org

If this is a brand-new account:

```
hermes orgs create --name "Acme Marketing"
# → created acme-marketing
```

Or list what you have access to and pick one:

```
hermes orgs list
```

## 3. Link this directory to the org

```
hermes link --org acme-marketing
hermes status         # confirm
```

After this, no command needs `--org` until you `unlink`.

## 4. Connect the data source

Pick the flag that matches your source — the `source_type` is derived from it:

```
hermes connections add --postgres-url 'postgres://readonly:...@host/db'
hermes connections add --mysql-url    'mysql://readonly:...@host:3306/db'
hermes connections add --csv-file     './users.csv'
hermes connections add --firestore-credentials "$(cat sa.json)" --firestore-project-id my-project
hermes connections add --posthog-host https://us.posthog.com --posthog-project-id 12345 --posthog-api-key phx_...
hermes connections add --stripe-api-key rk_live_...
```

The CLI introspects the schema and caches a snapshot — tables/columns for SQL sources, collections/inferred fields for Firestore, sampled events for PostHog, supported resources + live counts for Stripe. This snapshot is the **contract** the caller reads before writing any detection query.

You can add **more than one** source. The first becomes the org default; add later sources with `--default` to change which is the fallback. List them any time:

```
hermes connections list
hermes connections schema [<id>]   # omit <id> for the default source
```

The schema response leads with `source_type`, `query_language`, and an `example_query`. Read it before step 6. With multiple sources, note the connection id you intend to use and pass it through the rest of the flow. See [data-sources.md](../reference/data-sources.md) for dialect details.

## 5. Set up sending

This step requires DNS records to be set at the domain's registrar — apply them via a provider API, IaC, or hand them to the user, depending on what's available. See [set-up-sending.md](set-up-sending.md) for the focused workflow. Summary:

```
hermes domains add example.com         # prints DNS records
# ... human sets DNS records at the registrar ...
hermes domains verify example.com      # poll until 'verified'
hermes senders create --email hello@example.com --name "Acme Team"
```

## 6. Write and submit a trigger

Iterate on the detection query first, then submit:

```
# Test a candidate (read-only, no side effects)
hermes connections query "SELECT id, email FROM users WHERE created_at > now() - interval '24 hours' LIMIT 50"

# Looks right? Submit the trigger
hermes triggers create \
  --name "Welcome 24h signups" \
  --detection-query "SELECT id, email FROM users WHERE created_at > now() - interval '24 hours'" \
  --email-prompt "Write a warm, short welcome email mentioning the signup date naturally."
```

If you have more than one source, target one explicitly on both the query and the trigger: `hermes connections query "<q>" <id>` and `hermes triggers create --source <id> ...`. Omit them to use the org default.

For Firestore the query is a JSON DSL object instead of SQL — see [data-sources.md](../reference/data-sources.md) and [create-trigger.md](create-trigger.md) for the full authoring workflow.

## 7. Attach the sender + activate

```
hermes triggers update <trigger-id> --sender <sender-id>
hermes triggers activate <trigger-id>
```

## 8. Fire and review drafts

If the trigger has `auto_send` off (default), it creates drafts:

```
hermes triggers run <trigger-id>             # fire now
hermes drafts list --trigger <trigger-id>
hermes drafts show <draft-id>                # sanity-check a few bodies
hermes drafts approve <draft-id>             # send one
hermes drafts approve --all --trigger <trigger-id>   # or send all
```

See [review-drafts.md](review-drafts.md) for the full approve/reject workflow.

## 9. Watch results

```
hermes emails list --trigger <trigger-id>
hermes analytics campaign <trigger-id> --since 24h
```

## What to do next

- Add more triggers for other cohorts.
- Once you trust a trigger's quality, switch it to `auto_send`: `hermes triggers update <id> --auto-send`.
- Watch `hermes analytics overview` weekly to spot underperforming campaigns.
