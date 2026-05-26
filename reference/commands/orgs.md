# `hermes orgs`

Manage Hermes organizations.

## Fields

An organization maps 1:1 to a row in the `organizations` table.

| Column | Type | Default | Settable on `create` | Settable on `update` (flag) | Notes |
|---|---|---|---|---|---|
| `id` | uuid | auto | ✗ | ✗ | Primary key. |
| `owner_id` | uuid | session user | ✗ | ✗ | FK → `auth.users.id`. The user who created the org. |
| `role` | enum | — | ✗ | ✗ | Your membership role in this org (`owner` \| `admin` \| `member`). Joined, not a column on `organizations`. |
| `name` | text | — | `--name <name>` (required) | `--name <name>` | Human-readable org name. |
| `slug` | text | derived from `name` | `--slug <slug>` | ✗ (immutable) | URL-safe identifier. Unique globally. The agent-facing handle. |
| `logo_url` | text \| null | `null` | `--logo-url <url>` | `--logo-url <url>` | Brand logo, used in email templates. |
| `description` | text \| null | `null` | `--description <text>` | `--description <text>` | Short description shown in dashboards. |
| `from_name` | text \| null | `null` | `--from-name <name>` | `--from-name <name>` | Org-default From display name. Senders can override. |
| `reply_to` | text \| null | `null` | `--reply-to <addr>` | `--reply-to <addr>` | Org-default Reply-To. Senders can override. |
| `email_footer` | text \| null | `null` | `--email-footer <text>` | `--email-footer <text>` | Org-default footer appended to outbound emails. |
| `company_md` | text \| null | `null` | `--company-md <path-or-text>` | `--company-md <path-or-text>` | Markdown blob describing the company. Fed to the drafting agent as context. `@file` reads from a path. |
| `created_at` | timestamptz | now | ✗ | ✗ | |

The legacy `agentmail_*` columns and the dashboard-managed email-template state exist on the row but are **not** returned by the CLI API.

### Canonical JSON shape

```json
{
  "id": "org_...-uuid",
  "owner_id": "usr_...-uuid",
  "slug": "acme",
  "name": "Acme",
  "role": "owner",
  "logo_url": "https://...",
  "description": "B2B SaaS for widgets.",
  "from_name": "Acme Team",
  "reply_to": "support@acme.com",
  "email_footer": "Acme Inc. — 123 Market St, SF, CA",
  "company_md": "# Acme\n\nWe sell widgets to mid-market...",
  "created_at": "2026-04-01T10:00:00.000Z"
}
```

`orgs show` adds a `counts` object on top of this; `orgs list` returns a slimmer row (`id`, `slug`, `name`, `role`, `created_at`).

## Subcommands

> **Auth requirement.** All `orgs` commands require an **account-level CLI session** (`hermes auth login`). API keys (`hk_…`) are scoped to one org, so they can only see that one org and cannot create new ones — `orgs create` against an API key returns `FORBIDDEN` with a hint to use the device flow.

### `hermes orgs list`

Returns every org your account is a member of, with role.

```
$ hermes orgs list
SLUG            NAME             ROLE    ID
acme-marketing  Acme Marketing   owner   3f2c…-uuid
widgets         Widgets Inc.     member  9c1d…-uuid
```

### `hermes orgs create --name <name> [--slug <slug>]`

Creates a new org and makes the calling user the owner. `slug` is auto-generated from `name` if omitted (and must be globally unique). Returns the new org row.

```
hermes orgs create --name "Acme Marketing"
hermes orgs create --name "Acme Marketing" --slug acme
```

Optional flags map to columns as follows:

| Flag | Column written |
|---|---|
| `--name <name>` | `name` |
| `--slug <slug>` | `slug` |
| `--logo-url <url>` | `logo_url` |
| `--description <text>` | `description` |
| `--from-name <name>` | `from_name` |
| `--reply-to <addr>` | `reply_to` |
| `--email-footer <text>` | `email_footer` |
| `--company-md <path-or-text>` | `company_md` (if value starts with `@`, treats as file path) |

After create, run `hermes link --org <slug>` to operate against the new org.

### `hermes orgs show [--org <slug>]`

Returns the full row for the linked org (or `--org` override), plus counts of related resources:

```json
{
  "...": "<every column from the table above>",
  "counts": {
    "connections": 1,
    "domains": 2,
    "senders": 3,
    "triggers": 7,
    "drafts_pending": 4,
    "emails_sent_30d": 1240
  }
}
```

`<slug>` resolution: `--org acme` looks up `slug = 'acme'`. The `id` is also accepted.

### `hermes orgs update [flags] [--org <slug>]`

Updates the linked org (or `--org` override). Same flag → column mapping as `create`, minus `--slug` (slug is immutable after create). Returns the full updated row.

### `hermes orgs delete --confirm`

Deletes the linked org (or `--org` override). Owner-only, irreversible, always requires `--confirm`.

**The org must be empty.** Because the `organizations` row is referenced by connections, domains, senders, triggers, and emails without cascade, delete **refuses** (`CONFLICT`) if any of those still exist — it reports the counts and points you at the teardown commands. Remove the resources first (`connections remove`, `domains remove`, `senders delete`, `triggers delete`), or use the dashboard for a full wipe. After a successful delete, run `hermes unlink` to clear the stale local link.

## Notes for agents

- The natural bootstrap flow is **auth → orgs create → link → connections add → domains add → senders create → triggers create**. Every step is doable from the CLI now — no web onboarding required.
- After creating an org, run `hermes link --org <new-slug>` to operate on it.
- After deletion, the linked-project file in cwd becomes stale; run `hermes unlink --confirm`.
- `slug` is the agent-facing handle (stable, human-readable). Use it in `--org` and in `link`. The `id` works anywhere `slug` works, but slugs are preferred for log readability.
- The `agentmail_*` columns are legacy and will be `null` for new orgs. Don't write to them.
