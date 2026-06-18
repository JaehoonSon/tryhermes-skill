# API surface — driving Hermes via `hermesApi`

When you are the in-app assistant, you drive Hermes through the **`hermesApi`** tool instead of the `hermes` CLI. It takes `{ method, path, body? }` and calls the platform's own API, scoped to the current org. The command reference pages (`reference/commands/*.md`) describe the same operations — **a CLI flag is a body field** (e.g. `--email-prompt "…"` → `body.email_prompt`).

> Reads return immediately. Writes/deletes (create, edit, delete, approve, run, verify) **pause for the user's approval** automatically — issue the call; the platform surfaces an approval card. You do not need to ask first in text.

## Conventions
- Paths are relative to the API root, e.g. `GET /triggers`, `POST /drafts/<id>/approve`.
- Replace `<id>` / `<triggerId>` with a real id you obtained from a list/get call.
- Bodies are JSON objects using the **snake_case** field names from the command reference pages.
- List endpoints accept filters as a query string, e.g. `GET /drafts?status=drafted`.
- **Time windows accept a duration shorthand** — `30d`, `24h`, `2w`, `90d` — not just exact timestamps. Use it for `since` (and the overview `window_days`), e.g. `GET /analytics/overview?since=30d`, `GET /analytics/campaign/<id>?since=7d`, `GET /emails?since=24h`. `until` is an ISO timestamp and defaults to now. Prefer the shorthand; don't compute exact dates.

## Endpoint map

### Connections & data (read schema, iterate queries)
| Call | Purpose |
|---|---|
| `GET /connections` | List connected data sources. |
| `GET /connections/<id>` | One connection's details. |
| `GET /connections/<id>/schema` | Condensed schema (tables/collections + fields). |
| `POST /connections/<id>/query` body `{ "query": "SELECT … LIMIT 50" }` | Run a **read-only** query (SQL/HogQL string, or the Firestore/Stripe DSL object). Use to validate a cohort before creating a trigger. |
| `POST /connections/<id>/inspect-schema` | Force a fresh schema introspection. |

> **Multiple sources.** An org can have several connections (`GET /connections` lists them; one is the default). Pick the one matching the request, then use its `<id>` in the schema/query paths above and in `source_connection_id` on `POST /triggers`. When two sources are equally plausible and choosing wrong would build the wrong trigger, ask one concise question first. See [data-sources.md](data-sources.md).

> For authoring a trigger's detection query, the typed query tools (`testRemoteSqlQuery` etc.) are also available and render results inline — prefer them for the iterate-a-query loop; use `hermesApi` query for one-off checks. They accept a `connectionId` to target a non-default source.

### Signals (the discovery queue — read & triage)
| Call | Purpose |
|---|---|
| `GET /signals` (`?status=new\|reviewed\|dismissed`, `?sort=surprise\|recent`, `?include_dismissed=true`, `?limit=`) | List discovered signals. Default: open queue (new + reviewed), strongest first. Returns `{ signals, counts }`. |
| `GET /signals/<id>` | One signal's full row. |
| `PATCH /signals/<id>` body `{ "status": "reviewed" \| "dismissed" \| "new" }` | Triage only (no create/send). |
| `POST /signals/run` | Run a discovery pass. **Expensive — capped at once per 48h/org**; a `CONFLICT` means one ran recently or is running. Async: returns `{ job_id }`. |
| `GET /signals/run/<job_id>` | Poll a discovery pass: `{ state, signals_found?, error? }`. |

> A signal is a **finding, not machinery**: it has no detection query. To act on one, author a trigger for its audience (`POST /triggers`) using the signal's `who` / `insight` / `so_what` as the brief. See [signals.md](commands/signals.md).

### Triggers
| Call | Purpose |
|---|---|
| `GET /triggers` (`?status=active\|paused`) | List triggers. |
| `GET /triggers/<id>` | One trigger. |
| `POST /triggers` body `{ detection_query, email_prompt, source_connection_id?, … }` | Create a trigger. Pass `source_connection_id` to bind it to a specific source (defaults to the org default). (For the guided authoring flow, the typed `createIntentTrigger` tool with its preview card is preferred.) |
| `PATCH /triggers/<id>` body `{ …changed fields }` | Edit a trigger. |
| `DELETE /triggers/<id>` | Delete a trigger. |
| `POST /triggers/<id>/run` | Scan now — enqueues a run (returns a `job_id`; see polling below). |
| `POST /triggers/<id>/preview` | Non-persistent preview of detection. |

### Drafts (review & approve in chat)
| Call | Purpose |
|---|---|
| `GET /drafts` (`?status=drafted\|approved\|…`, `?trigger_id=<id>`) | List drafts awaiting review, optionally scoped to one trigger. |
| `GET /drafts/<id>` | One draft (subject, body, recipient). |
| `POST /drafts/<id>/approve` | Approve one draft → queues delivery. |
| `POST /drafts/<id>/reject` | Reject one draft → releases the recipient. |
| `POST /drafts/approve` body `{ trigger_id }` **or** `{ all_triggers: true, confirm: true }` | Bulk approve. |

### Emails (sent mail + delivery)
| Call | Purpose |
|---|---|
| `GET /emails` (`?status=sent\|delivered\|…`, `?trigger_id=<id>`, `?since=7d`) | List sent / in-flight emails, optionally scoped to one trigger. |
| `GET /emails/<id>` | One email + its delivery timeline (opens, clicks, bounces). |

### Analytics
| Call | Purpose |
|---|---|
| `GET /analytics/overview` (`?since=7d`) | Org-wide metrics (sends, opens, clicks, replies). |
| `GET /analytics/campaign/<triggerId>` (`?since=…`) | Per-trigger metrics. |

### Senders & domains (sending setup)
| Call | Purpose |
|---|---|
| `GET /senders` · `GET /senders/<id>` | List / show sender identities. |
| `POST /senders` · `PATCH /senders/<id>` · `DELETE /senders/<id>` | Create / update / remove a sender. |
| `GET /domains` · `GET /domains/<id>` | List / show sending domains + DNS records. |
| `POST /domains` · `POST /domains/<id>/verify` · `DELETE /domains/<id>` | Add / verify / remove a domain. |

## Async operations — poll for outcomes
`POST /triggers/<id>/run` and `POST /drafts/approve` are **asynchronous**: they enqueue work and return quickly (a `job_id`), not the final result. Do not claim emails were sent. To observe outcomes, poll:

1. After **run**: `GET /drafts?trigger_id=<id>` to see drafts the run produced.
2. After **approve**: `GET /emails?trigger_id=<id>` (and `GET /emails/<id>`) to confirm delivery, then `GET /analytics/campaign/<id>` for engagement.
3. After **signals discovery** (`POST /signals/run`): poll `GET /signals/run/<job_id>` until `state` is `completed`, then `GET /signals` to read what it surfaced. Don't claim new signals exist until the read returns them.

Tell the user it's processing and report concrete results once the follow-up reads return them.
