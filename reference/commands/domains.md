# `hermes domains`

Manage sending domains. A domain must be verified (DNS records set + propagated) before it can host senders.

## Fields

A domain maps 1:1 to a row in the `email_domains` table. Hermes maintains **three parallel verification tracks** per domain — the base sending identity, DMARC, and the reply subdomain — each with its own status and DNS records.

| Column | Type | Default | Settable on `create` | Settable on `update` (flag) | Notes |
|---|---|---|---|---|---|
| `id` | uuid | auto | ✗ | ✗ | Primary key. |
| `org_id` | uuid | session | ✗ | ✗ | Resolved from the linked org / `--org`. |
| `domain` | text | — | positional `<domain>` (required) | ✗ | The FQDN, e.g. `example.com`. Immutable; recreate to change. |
| `verification_status` | text | `"pending"` | ✗ | refreshed by `verify` | `pending` \| `verified` \| `failed`. Base SES identity. |
| `dns_records` | jsonb (array) | `[]` | ✗ | refreshed by `verify` | Records to set for base verification. Shape: `{ type, name, value, status? }`. |
| `dmarc_verification_status` | text | `"pending"` | ✗ | refreshed by `verify` | `pending` \| `verified` \| `failed`. |
| `dmarc_dns_records` | jsonb (array) | `[]` | ✗ | refreshed by `verify` | DMARC TXT record(s). |
| `reply_domain` | text | derived | ✗ | ✗ | The subdomain used for inbound replies (e.g. `reply.example.com`). Auto-derived on create. |
| `reply_domain_verification_status` | text | `"pending"` | ✗ | refreshed by `verify` | `pending` \| `verified` \| `failed`. |
| `reply_domain_dns_records` | jsonb (array) | `[]` | ✗ | refreshed by `verify` | MX + optionally TXT records for the reply subdomain. |
| `reply_routing_metadata` | jsonb \| null | `null` | ✗ | ✗ | `{ "mxValue": "...", "mxPriority": 10 }`. System-set. |
| `provider_identity_arn` | text \| null | `null` | ✗ | ✗ | AWS SES identity ARN. System-set. |
| `created_at` | timestamptz | now | ✗ | ✗ | |
| `updated_at` | timestamptz | now | ✗ | ✗ | |

A domain is considered **fully verified** only when `verification_status`, `dmarc_verification_status`, and `reply_domain_verification_status` are all `"verified"`. Senders can be created the moment `verification_status = "verified"`, but DMARC and reply routing are recommended before high-volume sending.

Every JSON response includes every column above.

### Canonical JSON shape

```json
{
  "id": "dom_...-uuid",
  "org_id": "org_...-uuid",
  "domain": "example.com",
  "verification_status": "pending",
  "dns_records": [
    { "type": "TXT",   "name": "_amazonses.example.com",    "value": "abc123...",                                     "status": "pending" },
    { "type": "CNAME", "name": "s1._domainkey.example.com", "value": "s1.dkim.amazonses.com",                         "status": "pending" },
    { "type": "CNAME", "name": "s2._domainkey.example.com", "value": "s2.dkim.amazonses.com",                         "status": "pending" }
  ],
  "dmarc_verification_status": "pending",
  "dmarc_dns_records": [
    { "type": "TXT", "name": "_dmarc.example.com", "value": "v=DMARC1; p=none; rua=mailto:dmarc@example.com", "status": "pending" }
  ],
  "reply_domain": "reply.example.com",
  "reply_domain_verification_status": "pending",
  "reply_domain_dns_records": [
    { "type": "MX", "name": "reply.example.com", "value": "10 feedback-smtp.us-east-1.amazonses.com", "status": "pending" }
  ],
  "reply_routing_metadata": { "mxValue": "feedback-smtp.us-east-1.amazonses.com", "mxPriority": 10 },
  "provider_identity_arn": null,
  "created_at": "2026-05-13T20:00:00.000Z",
  "updated_at": "2026-05-13T20:00:00.000Z"
}
```

## Subcommands

### `hermes domains list`

Returns an array of full domain rows.

### `hermes domains add <domain>`

Registers the domain with the email provider and populates `dns_records`, `dmarc_dns_records`, and `reply_domain_dns_records`. All three `*_verification_status` fields start at `pending`. Returns the full row.

```
hermes domains add example.com
```

### `hermes domains show <domain>`

Returns the full row. `<domain>` is the FQDN (e.g. `example.com`), not a slug or id.

### `hermes domains verify <domain>`

Triggers a verification check against DNS for all three tracks. Returns the full row with refreshed `*_verification_status` and per-record `status` values. Use after the human has added the records and DNS has propagated (usually 5–60 minutes).

### `hermes domains remove <domain> --confirm`

Removes the domain. Senders attached to it become unusable.

## Notes for agents

- `<domain>` in any command is the FQDN (`example.com`), not the `id`. Both are stable identifiers for the row.
- `add` returns DNS records that must be set at the domain's authoritative DNS provider before `verify` will succeed. The CLI itself does not modify DNS — apply the records via a provider API, IaC, or by handing them to the user, depending on what's available.
- DNS propagation is not instant. If `verify` returns any track as `pending`, wait 5–10 minutes and retry, up to ~1 hour. If still pending after that, compare DNS against the records `add` returned (typos in TXT values are common).
- Inspect each track individually — it's normal for `verification_status` to flip to `verified` minutes before `dmarc_verification_status` does.
- Multiple domains per org are supported — useful for sending different campaigns from different brands.

## Common errors

| Code | Meaning | Hint |
|---|---|---|
| `DOMAIN_ALREADY_EXISTS` | Already added to this org | Use `hermes domains show <domain>` to see records |
| `DOMAIN_OWNED_BY_OTHER_ORG` | Another Hermes org claimed it | Contact support — domains are exclusive |
| `DNS_NOT_PROPAGATED` | `verify` ran but records aren't visible yet | Wait and retry; check records match exactly |
