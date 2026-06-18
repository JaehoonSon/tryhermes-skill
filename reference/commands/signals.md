# `hermes signals`

Read and triage the strategist's **signal queue** — a ranked list of non-obvious, audience-centered discoveries about this org's customers. A signal answers *who* a hidden pattern is about, *why* a dashboard would have missed it, and *what's at stake*. It is an **insight to act on, not an action**: a signal carries no detection query and sends nothing.

> **Signals vs. triggers.** A trigger is machinery (detection query + email prompt) that runs a campaign. A signal is a finding. Acting on a signal means *you* read it, decide it's worth a campaign, and author a trigger for that audience yourself (see [create-trigger.md](../workflows/create-trigger.md)). Nothing converts a signal to a trigger automatically.

## Fields

A signal maps 1:1 to a row in the `signals` table. Every `list` and `show` response returns the full row (signals are compact text, so there's no reduced list shape).

| Column              | Type        | Notes                                                                                      |
| ------------------- | ----------- | ------------------------------------------------------------------------------------------ |
| `id`                | uuid        | Primary key. Use wherever the docs say `<id>`.                                              |
| `headline`          | text        | The aha in one plain sentence ("you assume X, but actually Y").                            |
| `who`               | text        | The cohort the signal is about, named by the behavior that defines it.                     |
| `audience_size`     | text        | Rough human-scale size ("~120 accounts", "1 in 6 teams").                                  |
| `insight`           | text        | The non-obvious thing that's true about this audience, with the number.                    |
| `why_hidden`        | text        | Why a normal dashboard / gut-check would have missed it.                                    |
| `so_what`           | text        | The value or risk at stake, framed as an insight (not an action).                          |
| `surprise`          | integer     | How non-obvious it is: 0 = already knew, 100 = never would have found it.                  |
| `confidence`        | integer     | How sure it's real (0–100), orthogonal to `surprise`.                                       |
| `status`            | text        | `new` → `reviewed` → `dismissed` (and back). `dismissed` is hidden from the open queue.    |
| `observation_count` | integer     | How many discovery runs re-found this signal. High = well established.                      |
| `last_observed_at`  | timestamptz | When a run last re-observed it.                                                             |
| `created_at`        | timestamptz | First surfaced.                                                                            |

### Canonical JSON shape

```json
{
  "id": "sig_...-uuid",
  "headline": "Your stickiest accounts are the ones onboarding ignored",
  "who": "Teams that skipped the setup wizard but invited 3+ members in week one",
  "audience_size": "~90 accounts",
  "insight": "They retain at 2.1× the median despite never finishing onboarding.",
  "why_hidden": "Onboarding dashboards only track wizard completion, so these look like drop-offs.",
  "so_what": "A 'completion = activation' model is mis-scoring your best cohort.",
  "surprise": 82,
  "confidence": 74,
  "status": "new",
  "observation_count": 3,
  "last_observed_at": "2026-06-14T09:02:11.000Z",
  "created_at": "2026-05-30T09:01:55.000Z"
}
```

## Commands

### `hermes signals list [--status new|reviewed|dismissed] [--sort surprise|recent] [--include-dismissed] [--limit <n>]`

Returns the signal queue. Defaults: the **open** queue (new + reviewed, dismissed excluded), strongest discovery first (`--sort surprise`). Use `--sort recent` to order by `last_observed_at`. `--status` filters to exactly one status. `--include-dismissed` adds dismissed signals when no `--status` is set. `--limit` caps the count (1–200, default 100). The non-JSON output also prints queue counts (open / new / reviewed / dismissed).

### `hermes signals show <id>`

Prints one signal's full row.

### `hermes signals review <id>` / `hermes signals dismiss <id>` / `hermes signals restore <id>`

Triage only — these set `status` and create nothing:

- `review` → `reviewed` (you've read it / acted on it; keeps it in the open queue)
- `dismiss` → `dismissed` (drop it from the open queue)
- `restore` → `new` (un-dismiss)

### `hermes signals discover`

Kick off a fresh discovery pass: the strategist agent scans this org's connected data for new hidden audiences and writes any it finds to the queue.

- **Expensive — capped at once every 2 days per org.** If a pass ran (or is still running) within the last 48 hours, this returns `CONFLICT`; read the existing queue with `hermes signals list` instead. The cooldown is enforced off the live job queue, so it can't be dodged by a run that surfaced nothing.
- **Async.** Returns a `job_id`. By default the CLI polls until the pass finishes, then reports how many signals were surfaced. Pass the global `--no-wait` to return the `job_id` immediately instead.

## Driving it via `hermesApi` / MCP

Same operations, REST-style (a CLI flag is a query param or body field):

| Intent                | CLI                       | REST (`hermesApi` / MCP tool)                         |
| --------------------- | ------------------------- | ----------------------------------------------------- |
| List the queue        | `signals list`            | `GET /signals?status=&sort=&include_dismissed=&limit=` · `list_signals` |
| Read one              | `signals show <id>`       | `GET /signals/<id>` · `get_signal`                    |
| Triage                | `signals review/dismiss`  | `PATCH /signals/<id>` `{ "status": "reviewed" }` · `update_signal` |
| Run discovery         | `signals discover`        | `POST /signals/run` · `discover_signals`              |
| Poll a discovery run  | (auto)                    | `GET /signals/run/<job_id>` · `get_discovery_run`     |

## Notes for agents

- **Read the queue before discovering.** `signals list` is free and instant; `signals discover` is expensive and rate-limited. Only run discovery when the queue is stale or empty and you genuinely need fresh ground.
- **A signal is a lead, not a command.** Decide whether it's worth acting on. If it is, author a trigger for that audience (read the source schema, write the detection query — the signal won't give you one). If it isn't, `dismiss` it.
- **Triage what you touch.** Mark a signal `reviewed` once you've acted on or evaluated it so the open queue reflects what's actually outstanding.
- **`observation_count` is a staleness hint.** A high count means the org has seen this signal across many runs — push for newer, less-obvious ground rather than re-surfacing it.

## Common errors

| Code              | Meaning                                              | Hint                                                              |
| ----------------- | ---------------------------------------------------- | ---------------------------------------------------------------- |
| `CONFLICT`        | A discovery pass ran (or is running) within 48h      | Read the queue with `hermes signals list`; wait for the cooldown |
| `NOT_FOUND`       | No signal with that id in this org                   | Check the id with `hermes signals list`                          |
| `VALIDATION_FAILED` | Bad `--status`, `--sort`, or `--limit` value       | Use the documented enum / range                                  |
