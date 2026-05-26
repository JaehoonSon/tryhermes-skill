# `hermes senders`

Manage email sender identities. A sender is an address (`local_part@verified-domain`) plus display metadata, bound to one verified domain.

## Fields

A sender maps 1:1 to a row in the `email_sender_identities` table.

| Column | Type | Default | Settable on `create` | Settable on `update` (flag) | Notes |
|---|---|---|---|---|---|
| `id` | uuid | auto | ✗ | ✗ | Primary key. |
| `org_id` | uuid | session | ✗ | ✗ | Resolved from the linked org / `--org`. |
| `domain_id` | uuid \| null | derived from `--email` | ✗ | ✗ | FK → `email_domains.id`. System-set from the email's domain part. |
| `provider_connection_id` | uuid \| null | `null` | `--provider-connection <id>` | `--provider-connection <id>` | FK → `email_provider_connections.id`. Only set for `kind = "connected_mailbox"`. |
| `kind` | text | `"custom"` | `--kind <platform\|custom\|connected_mailbox>` | ✗ | Identity kind. Immutable after create. |
| `provider` | text | `"ses"` | `--provider <ses\|microsoft_graph>` | ✗ | Sending provider. Immutable after create. |
| `email_address` | text | — | `--email <addr>` (required) | ✗ | The full address. Immutable — create a new sender to change. Unique per org. |
| `local_part` | text \| null | derived from `--email` | ✗ | ✗ | Local part of the email. System-set. |
| `from_name` | text \| null | `null` | `--name <display-name>` (alias `--from-name`) | `--name <display-name>` (alias `--from-name`) | Display name shown in clients. |
| `reply_to` | text \| null | `null` | `--reply-to <addr>` | `--reply-to <addr>` | Optional Reply-To override. |
| `verification_status` | text | `"pending"` | ✗ | ✗ | `pending` \| `verified` \| `failed`. Tracks provider-side identity verification. |
| `is_default` | boolean | `false` | `--default` | `--default` (promote only) | Marks the org's default sender. Unique per org — setting it on one row unsets the previous (single txn). There is no demote flag: to change the default, promote a different sender. |
| `is_enabled` | boolean | `true` | `--enabled` / `--no-enabled` | `--enabled` / `--no-enabled` | If `false`, triggers referencing this sender will fail to send. |
| `created_at` | timestamptz | now | ✗ | ✗ | |
| `updated_at` | timestamptz | now | ✗ | ✗ | |

Every JSON response includes every column above.

### Canonical JSON shape

```json
{
  "id": "snd_...-uuid",
  "org_id": "org_...-uuid",
  "domain_id": "dom_...-uuid",
  "provider_connection_id": null,
  "kind": "custom",
  "provider": "ses",
  "email_address": "hello@example.com",
  "local_part": "hello",
  "from_name": "Acme Team",
  "reply_to": null,
  "verification_status": "verified",
  "is_default": true,
  "is_enabled": true,
  "created_at": "2026-05-13T20:00:00.000Z",
  "updated_at": "2026-05-13T20:00:00.000Z"
}
```

## Subcommands

### `hermes senders list`

Returns an array of full sender rows. To filter to a single domain pass `--domain <fqdn>` (e.g. `--domain example.com`). The server resolves the FQDN to the underlying `domain_id`; you never need to look the UUID up yourself.

### `hermes senders create --email <addr> --name <display-name>`

Creates a sender. The domain part of `--email` must be a verified domain on this org (`verification_status = "verified"` on the matching `email_domains` row).

```
hermes senders create --email hello@example.com --name "Acme Team"
```

Recipients see `From: Acme Team <hello@example.com>`. Returns the full row.

Optional flags map to columns as follows:

| Flag | Column written |
|---|---|
| `--email <addr>` | `email_address` (also derives `domain_id` and `local_part`) |
| `--name <display>` (alias `--from-name`) | `from_name` |
| `--reply-to <addr>` | `reply_to` |
| `--kind <platform\|custom\|connected_mailbox>` | `kind` |
| `--provider <ses\|microsoft_graph>` | `provider` |
| `--provider-connection <id>` | `provider_connection_id` |
| `--default` | `is_default` (set true; unsets any other default in the org) |
| `--enabled` / `--no-enabled` | `is_enabled` |

### `hermes senders show <id>`

Returns the full sender row, plus a `triggers` array listing trigger ids currently referencing this sender via `sender_identity_id`.

### `hermes senders update <id> [flags]`

`email_address`, `local_part`, `domain_id`, `kind`, and `provider` are immutable. Mutable columns and their flags:

| Flag | Column written |
|---|---|
| `--name <new-name>` (alias `--from-name`) | `from_name` |
| `--reply-to <addr>` (use `--reply-to ""` to clear) | `reply_to` |
| `--provider-connection <id>` | `provider_connection_id` |
| `--default` | promotes this sender to org default (unsets the prior default in the same txn). Promote-only — there is no `--no-default`. |
| `--enabled` / `--no-enabled` | `is_enabled` |

Returns the full updated row.

### `hermes senders delete <id> --confirm`

Triggers that reference this sender (via `sender_identity_id`) will fail to send until reassigned via `hermes triggers update <trigger-id> --sender <other-sender-id>`.

## Notes for agents

- Many orgs only need one sender (e.g., `hello@theirdomain.com`). Don't proliferate senders unless the user asks.
- `from_name` matters — it's what most email clients show prominently. Use the brand name, not a person's name, unless the user specifies.
- The org default (`is_default = true`) is at most one row per org — setting it elsewhere automatically unsets the previous default. **There is no demote operation.** To "change the default," promote a different sender; the prior default is unset in the same transaction. Demoting without promoting would leave the org with zero defaults, which breaks any code path that relies on the org default — so the API rejects `is_default: false` outright.
- **You never need to know a domain's UUID to create a sender.** `--email hello@example.com` is the only input — the domain is derived from the address. Use `hermes domains list` to discover the FQDN if you don't have it; never pass `domain_id` directly.

## Common errors

| Code | Meaning | Hint |
|---|---|---|
| `DOMAIN_NOT_VERIFIED` | Domain part of `email_address` isn't verified | Run `hermes domains verify <domain>` first |
| `SENDER_IN_USE` | Cannot delete; triggers reference it via `sender_identity_id` | Reassign with `hermes triggers update <id> --sender <other-id>` |
| `INVALID_EMAIL` | `email_address` is malformed | Use a standard `local@domain` format |
| `DUPLICATE_SENDER` | A sender with this `email_address` already exists in the org | `email_address` is unique per org |
