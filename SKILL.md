---
name: hermes
description: Use when interacting with the Hermes hyperpersonalization platform via the `hermes` CLI — creating campaign triggers from natural-language intent, managing customer database connections, sending domains and email senders, reviewing and approving email drafts, and inspecting campaign analytics. Trigger on tasks like "create a trigger to email power users", "connect our customer DB", "verify our sending domain", "approve this campaign's drafts", or "show last week's open rate".
---

# Hermes

Hermes is an email automation platform: you connect a data source, write a detection query that selects a user cohort, give a brief email prompt, and Hermes drafts and (with approval) sends personalized emails per user.

You drive Hermes via the `hermes` CLI. **You are the reasoning loop.** Hermes is the platform: it validates your query against the live source, persists triggers, runs the scan/draft/send pipeline, and reports back. There is no chat agent on the CLI side — you read the schema, you write the query, you submit the trigger.

This skill teaches you the mental model and points you at the right command or workflow for each task.

> **Status:** This skill describes the planned CLI surface. Implementation may not yet be complete; verify with `hermes --help` before assuming a command works.

## Mental model

```
organization
  ├── connection           (one or more customer data sources — Postgres, MySQL, CSV, Firestore, PostHog, Stripe; one is the default)
  ├── domain → sender      (a verified domain, then one or more sender identities on it)
  └── trigger              (detection query + email prompt, both caller-written; bound to one source)
       ├── draft           (one per detected user, per run, awaiting approval)
       └── email           (a sent or in-flight message)
```

Every command operates on the **linked organization** (set via `hermes link`) unless overridden with `--org`.

## Multiple data sources

An org can connect **several** data sources at once (e.g. a production Postgres, an analytics PostHog, a Stripe billing account, a CSV upload). One is the **default**, but the default is only a fallback — not a reason to be stubborn.

- **List before you pick.** When the org has more than one source, run `hermes connections list` and choose the source that matches the request (a named database, an analytics system, an uploaded file, a product area).
- **Thread one source through the whole flow.** Once you pick a connection id, pass it to every step: `connections schema <id>`, `connections query "<q>" <id>`, and `triggers create --source <id>`. A trigger is bound to a single source via `source_connection_id`.
- **Each source speaks its own dialect.** Postgres/MySQL/CSV are SQL, Firestore and Stripe are JSON DSLs, PostHog is HogQL. Read the chosen source's schema first — see [data-sources.md](reference/data-sources.md).
- **When to ask.** If two sources are equally plausible and picking wrong would build the wrong trigger, ask one concise question. Otherwise pick the best match and say which source you used.

## Required setup before sending

In order: **auth → (orgs create) → link → connection → domain → sender → trigger**.

`auth` accepts two paths:
- **Account-level** (default): `hermes auth login` opens a browser, signs you in, and stores a `cli_…` session token that can act on any org you're a member of. Pair with `hermes link --org <slug>`.
- **Org-scoped API key** (CI / automation): `hermes auth login --key hk_…` uses a key minted in the dashboard, bound to one org.

`orgs create` is only meaningful with account-level auth — API keys can't create new orgs. New orgs come with no resources; you bootstrap them via `connections add` → `domains add` → `senders create` → `triggers create`, all from the CLI.

You cannot send email without a verified domain plus a sender on it. You cannot write a useful detection query without a connection. The CLI surfaces helpful errors with hints when steps are skipped.

## Authoring a trigger (the core loop)

```
hermes connections list                            # (multi-source orgs) pick the source that matches the request
hermes connections schema [<id>]                   # read source_type + dialect + example query
hermes connections query "<candidate>" [<id>]      # iterate read-only until the cohort is right
hermes triggers create \
  --detection-query "<validated query>" \
  --email-prompt "..." \
  [--source <id>] [...other flags]                 # submit; Hermes re-validates against the live source
```

Omit `<id>` / `--source` to use the org default. With multiple sources, pass the **same** connection id to every step so the schema you read, the query you test, and the trigger you create all target one source.

The detection query is **not always SQL** — for Firestore and Stripe connections it's a JSON DSL object. Always read the connection's schema first to learn the right dialect. See [data-sources.md](reference/data-sources.md).

## Where to look

**Workflows (task-oriented):**
- [Getting started end-to-end](workflows/getting-started.md)
- [Create a trigger from natural language](workflows/create-trigger.md)
- [Set up a sending domain and sender](workflows/set-up-sending.md)
- [Review and send drafts](workflows/review-drafts.md)

**Command reference (one file per family):**
- [auth](reference/commands/auth.md), [link](reference/commands/link.md), [orgs](reference/commands/orgs.md)
- [connections](reference/commands/connections.md), [domains](reference/commands/domains.md), [senders](reference/commands/senders.md)
- [triggers](reference/commands/triggers.md), [drafts](reference/commands/drafts.md), [emails](reference/commands/emails.md)
- [analytics](reference/commands/analytics.md)

**Background:**
- [Data model](reference/data-model.md)
- [Data sources: dialects, schema shapes, example queries](reference/data-sources.md)
- [Conventions: flags, output, async ops, errors](reference/conventions.md)
- [API surface — driving Hermes via `hermesApi` (in-app assistant)](reference/api-surface.md)

> **Two ways to drive Hermes.** In a terminal you use the `hermes` CLI. As the **in-app assistant** you use the `hermesApi` tool (`{ method, path, body }`) — same operations, REST-style. See [api-surface.md](reference/api-surface.md) for the path/body map. The command pages below describe the semantics either way (a CLI flag is a body field).
