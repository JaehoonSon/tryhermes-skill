# Data model

The conceptual entities you'll work with via the CLI. Each entity maps 1:1 to a Postgres table; the per-command reference pages list every column. This page is the high-level map.

## One rule: rows in, rows out

Every command operates on a row from one of the tables below. JSON output is the row, with **every column** present and named exactly as the database column (snake_case). See [conventions.md](conventions.md#field-shapes) for the field-shape policy.

## Organization → `organizations`

Top-level tenant. One Hermes org per business / service / campaign-bundle.

- Owns: connections, domains, senders, triggers, drafts, emails.
- Identified by `slug` (e.g. `acme-marketing`) or `id` (UUID).
- Brand defaults (`from_name`, `reply_to`, `email_footer`, `company_md`) live on this row and are inherited by triggers/senders unless overridden.
- Full columns: [orgs.md → Fields](commands/orgs.md#fields).

## Connection → `connections`

The customer's data source that Hermes watches.

- One per org (currently).
- `source_type`: one of `postgres`, `mysql`, `csv`, `firestore`. The first three are SQL-family (CSV is loaded into a managed Postgres and queried as such); Firestore is a NoSQL document store with its own JSON DSL. See **[data-sources.md](data-sources.md)** for the full per-source matrix — query language, schema snapshot shape, dialect caveats, example detection queries.
- `status`: `pending` → `connected` → `synced` → `failed`.
- `schema_snapshot` is what the caller reads to discover the source's tables/collections, dialect, and a literal example query for that source. It's the contract — always read it before writing a `detection_query`.
- Secret columns (`postgres_url`, `mysql_url`, `firestore_credentials`) are encrypted at rest and returned as `"<redacted>"` in CLI output.
- Full columns: [connections.md → Fields](commands/connections.md#fields).

## Domain → `email_domains`

A DNS domain owned by the org, used as the sender's domain.

- Identified by `domain` (FQDN, e.g. `example.com`) or `id`.
- **Three parallel verification tracks**, each with its own status + DNS records column:
  - `verification_status` + `dns_records` — base SES identity
  - `dmarc_verification_status` + `dmarc_dns_records` — DMARC
  - `reply_domain_verification_status` + `reply_domain_dns_records` — inbound reply subdomain
- Verification requires DNS records (returned on add) to be set at the customer's DNS provider — Hermes prints them; **a human must set them**.
- One org can verify multiple domains.
- Full columns: [domains.md → Fields](commands/domains.md#fields).

## Sender → `email_sender_identities`

An email sender identity — `email_address` plus display metadata, bound to one verified domain.

- `email_address` is unique per org and immutable. `from_name` and `reply_to` are mutable.
- `kind`: `platform` | `custom` | `connected_mailbox`. `provider`: `ses` | `microsoft_graph`.
- `is_default = true` is unique per org (one default sender at a time).
- `is_enabled = false` disables sending without deleting the row.
- Triggers reference one sender via `sender_identity_id`.
- Full columns: [senders.md → Fields](commands/senders.md#fields).

## Trigger → `triggers`

A campaign definition: an audience (who) plus an intent (what to say).

- Created via natural language: Hermes translates intent → `detection_query` (SQL) + `email_prompt`.
- Content fields (`detection_query`, `email_prompt`) are iteratively edited via `hermes triggers refine`, not literal flags.
- Operational columns (settable via flags on `update`): `auto_send`, `cooldown_hours`, `is_recurring`, `recurrence_cooldown_days`, `scan_interval_minutes`, `category`, `footer_enabled`, `footer_override`, `address_enabled`, `address_override`, `track_opens`, `track_clicks`.
- `is_active` toggled by `activate` / `pause`.
- Full columns: [triggers.md → Fields](commands/triggers.md#fields).

## Draft & Email → `emails`

Drafts and sent emails share the **same table** — `hermes drafts` and `hermes emails` are just different status filters on it.

- `status` column drives the split:
  - Drafts: `drafted` → `approved` (queued for send) | `rejected` (claim released; user re-eligible).
  - Emails: `sent` → `delivered` → `opened` / `clicked`, or terminal `bounced` / `complained`.
- One row per (trigger, detected user) per fire.
- `user_row_snapshot` captures the customer-DB row used at draft time, enabling faithful regeneration.
- Full columns: [drafts.md → Fields](commands/drafts.md#fields).

## Lifecycle

```
trigger fires (manual or scheduled)
  → detection_query runs against connection
    → for each new user: claim is recorded
      → AI drafts email (row inserted into `emails` with status='drafted')
        → if trigger.auto_send: status → 'approved' → send → status updates via webhooks
        → else:                 human approves → status='approved' → send
                                human rejects  → status='rejected', claim released, user re-eligible
```

## Identifier conventions

| Entity | Primary id | Agent-friendly handle |
|---|---|---|
| Organization | `id` (uuid) | `slug` (preferred in `--org`, `link`) |
| Connection | `id` (uuid) | — |
| Domain | `id` (uuid) | `domain` (FQDN; accepted everywhere `id` is) |
| Sender | `id` (uuid) | — |
| Trigger | `id` (uuid) | — |
| Draft / Email | `id` (uuid) | — |

When in doubt, agents should use `id` — it's stable, unambiguous, and accepted by every command.
