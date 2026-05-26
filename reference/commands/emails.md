# `hermes emails`

Read-only commands for sent and in-flight emails. For pre-send drafts, use [`hermes drafts`](drafts.md).

`emails` and `drafts` share the same `emails` table. `hermes emails` returns rows where `status` is past `approved` — i.e. one of: `sent`, `delivered`, `opened`, `clicked`, `bounced`, `complained`.

## Fields

A row maps 1:1 to a row in the `emails` table — same shape as documented in [drafts.md → Fields](drafts.md#fields). All columns are returned on every response. Repeated here for self-contained reference:

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key. |
| `org_id` | uuid | Owning org. |
| `trigger_id` | uuid \| null | FK → `triggers.id`. Null if the originating trigger was later deleted. |
| `recipient_email` | text | |
| `recipient_external_id` | text \| null | The user's id in the customer DB. |
| `subject` | text | |
| `body_html` | text | |
| `status` | text | One of `sent` \| `delivered` \| `opened` \| `clicked` \| `bounced` \| `complained`. |
| `sender_identity_id` | uuid \| null | FK → `email_sender_identities.id`. |
| `email_thread_id` | uuid \| null | FK → `email_threads.id`. |
| `email_message_id` | uuid \| null | FK → `email_messages.id`. |
| `ses_message_id` | text \| null | SES provider message id. |
| `agentmail_message_id` | text \| null | Legacy provider id. |
| `agentmail_thread_id` | text \| null | Legacy provider id. |
| `opened_at` | timestamptz \| null | First open event. |
| `clicked_at` | timestamptz \| null | First click event. |
| `sent_at` | timestamptz \| null | Send completed. |
| `unsubscribe_token` | text \| null | Token used in unsubscribe URLs. |
| `user_row_snapshot` | jsonb \| null | Snapshot of the user row used to draft this email. |
| `created_at` | timestamptz | Row creation time (draft time). |

### Canonical JSON shape

```json
{
  "id": "em_...-uuid",
  "org_id": "org_...-uuid",
  "trigger_id": "tr_...-uuid",
  "recipient_email": "alice@example.com",
  "recipient_external_id": "u_internal_abc",
  "subject": "Welcome to Acme",
  "body_html": "<p>Hi Alice...</p>",
  "status": "opened",
  "sender_identity_id": "snd_...-uuid",
  "email_thread_id": "thr_...-uuid",
  "email_message_id": "msg_...-uuid",
  "ses_message_id": "01000180a3...",
  "agentmail_message_id": null,
  "agentmail_thread_id": null,
  "opened_at": "2026-05-13T20:15:00.000Z",
  "clicked_at": null,
  "sent_at": "2026-05-13T20:12:00.000Z",
  "unsubscribe_token": "tok_...-base64",
  "user_row_snapshot": { "id": "u_internal_abc", "email": "alice@example.com", "signup_at": "2026-05-13T18:00:00Z" },
  "created_at": "2026-05-13T20:11:04.123Z"
}
```

## Subcommands

### `hermes emails list [filters]`

Returns an array of full email rows.

| Flag | Column filtered |
|---|---|
| `--trigger <id>` | `trigger_id` |
| `--status sent\|delivered\|opened\|clicked\|bounced\|complained` | `status` |
| `--since <duration>` | `sent_at` ≥ now − duration (e.g. `7d`, `24h`) |
| `--recipient <email>` | `recipient_email` |

### `hermes emails show <id>`

Returns the full row, plus the status timeline reconstructed from `email_events` (provider webhook events: bounces, complaints, opens, clicks, replies) and any inbound replies linked via `email_thread_id`.

```json
{
  "...": "<every column from the table above>",
  "events": [
    { "type": "sent",      "at": "2026-05-13T20:12:00.000Z" },
    { "type": "delivered", "at": "2026-05-13T20:12:03.412Z" },
    { "type": "opened",    "at": "2026-05-13T20:15:00.000Z" }
  ],
  "replies": [
    { "id": "msg_...", "from": "alice@example.com", "received_at": "2026-05-13T20:30:00.000Z", "preview": "Thanks!" }
  ]
}
```

## Notes

- Email status updates arrive via webhooks from the email provider (SES). Status can lag by seconds to minutes after the underlying event.
- `hermes emails` does **not** include drafts (status `drafted`, `approved`, `rejected`) — use [`hermes drafts list`](drafts.md) for those.
- `hermes emails list --recipient <email>` is the right way to answer "what have we ever sent this person?" across all triggers.
- The same `id` works in both `hermes drafts show` and `hermes emails show` — which one is "valid" depends only on the current `status` of the row.
