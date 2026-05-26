# `hermes drafts`

The pre-send queue. A draft is created per detected user when a trigger fires, *unless* the trigger has `auto_send` enabled.

Drafts and emails share the same underlying table (`emails`); `hermes drafts` operates on rows where `status IN ('drafted', 'approved', 'rejected')`, while [`hermes emails`](emails.md) operates on rows where `status` has progressed past `approved` (i.e. `sent`, `delivered`, `opened`, `clicked`, `bounced`, `complained`).

## Fields

| Column | Type | Notes |
|---|---|---|
| `id` | uuid | Primary key. |
| `org_id` | uuid | Owning org. |
| `trigger_id` | uuid \| null | FK → `triggers.id`. Nullable because historical rows survive trigger deletion (the FK is detached on delete). |
| `trigger_name` | text \| null | Convenience join of the trigger's name (not a column on `emails`). Null if the trigger was deleted. |
| `recipient_email` | text | The destination address. |
| `recipient_external_id` | text \| null | The user's id in the customer DB (from the detection query). |
| `subject` | text | Drafted subject line. |
| `body_html` | text | Drafted HTML body. |
| `body_text` | text \| null | Drafted plain-text body — a real stored column, not derived from the HTML. The CLI's human output renders this; `--json` returns both. |
| `body_format` | text | `"html"` or `"text"` — the format the email was drafted in. |
| `status` | text | For drafts: `drafted` \| `approved` \| `rejected`. After send, transitions into the lifecycle covered by [`hermes emails`](emails.md). |
| `sender_identity_id` | uuid \| null | FK → `email_sender_identities.id`. Set from the trigger's sender at draft time. |
| `email_thread_id` | uuid \| null | FK → `email_threads.id`. Populated when this draft is associated with a thread. |
| `email_message_id` | uuid \| null | FK → `email_messages.id`. Set after send. The `emails show` timeline is keyed off this. |
| `ses_message_id` | text \| null | Provider message id, set after send. |
| `opened_at` | timestamptz \| null | First open event timestamp. Null for unsent drafts. |
| `clicked_at` | timestamptz \| null | First click event timestamp. Null for unsent drafts. |
| `sent_at` | timestamptz \| null | Sent timestamp. Null for unsent drafts. |
| `unsubscribe_token` | text \| null | Token used in unsubscribe URLs. Set at send time. |
| `user_row_snapshot` | jsonb \| null | Snapshot of the user row used to draft this email. Enables faithful regeneration without re-running the trigger detection query. |
| `created_at` | timestamptz | Draft creation time. |

Every JSON response includes every column above.

### Canonical JSON shape

```json
{
  "id": "em_...-uuid",
  "org_id": "org_...-uuid",
  "trigger_id": "tr_...-uuid",
  "trigger_name": "Welcome 24h signups",
  "recipient_email": "alice@example.com",
  "recipient_external_id": "u_internal_abc",
  "subject": "Welcome to Acme",
  "body_html": "<p>Hi Alice...</p>",
  "body_text": "Hi Alice...",
  "body_format": "html",
  "status": "drafted",
  "sender_identity_id": "snd_...-uuid",
  "email_thread_id": null,
  "email_message_id": null,
  "ses_message_id": null,
  "opened_at": null,
  "clicked_at": null,
  "sent_at": null,
  "unsubscribe_token": null,
  "user_row_snapshot": { "id": "u_internal_abc", "email": "alice@example.com", "signup_at": "2026-05-13T18:00:00Z" },
  "created_at": "2026-05-13T20:11:04.123Z"
}
```

## Subcommands

### `hermes drafts list [filters]`

Returns an array of full draft rows.

| Flag | Column filtered |
|---|---|
| `--trigger <id>` | `trigger_id` |
| `--status drafted\|approved\|rejected` | `status` (default: `drafted`) |
| `--recipient <email>` | `recipient_email` |

Default: rows where `status = 'drafted'` across all triggers in the linked org — the actionable queue.

### `hermes drafts show <id>`

Human output renders a header (recipient, subject, status, trigger) plus the plain-text body. `--json` returns the full row (both `body_html` and `body_text`).

### `hermes drafts approve <id>`

Sets `status = 'approved'` and enqueues a send job on the `email-delivery` queue. **Returns immediately** — the send happens asynchronously in the worker; the CLI does not block or poll. Check the outcome later with `hermes emails show <id>` (same id), where `sent_at`, `ses_message_id`, and the delivery timeline appear once the worker processes it.

Response: `{ "approved": true, "id": "...", "status": "approved" }`. Errors `409` if the row isn't `drafted` (already approved/sent), `404` if it doesn't exist.

### Bulk approve

```
hermes drafts approve --all --trigger <id>      # every drafted row for one trigger
hermes drafts approve --all-triggers --confirm  # every drafted row in the org (guarded)
```

Trigger-scoped bulk requires `--trigger`. The org-wide form requires both `--all-triggers` and `--confirm` so it can't fire by accident. Returns `{ "approved_count": <n>, "ids": [...] }`; all approved rows are enqueued for delivery.

### `hermes drafts reject <id>`

Sets `status = 'rejected'`. **Important side effect:** the trigger's claim on this user is released (its `trigger_events` row is deleted), so the user becomes eligible for re-detection on the next trigger run. Returns `{ "rejected": true, "id": "...", "claim_released": true }`.

This is the right path when:
- The draft isn't quite right and you want Hermes to try again later (likely with refined trigger settings).
- You want this user to receive *some* email from this trigger, just not this one.

This is **not** the right path when the trigger itself is wrong — pause the trigger (or recreate it with a better query); don't reject drafts one by one.

## Notes for agents

- Default `drafts list` returns only `status = 'drafted'` so the queue is the actionable surface. Pass an explicit `--status` to inspect history.
- Always read `hermes drafts show <id>` before approving in agent flows. AI-drafted bodies are usually correct, but per-user personalization can occasionally hallucinate (wrong feature name, wrong tone for a high-value account). Cost of inspection is low; cost of one bad email to a key user is high.
- For non-trivial campaigns (>~20 drafts), sample 3–5 with `show`, then bulk approve if quality is good.
- `drafts` and `emails` share the same row — once you `approve` and the send completes, the same `id` is queryable via `hermes emails show <id>`.
