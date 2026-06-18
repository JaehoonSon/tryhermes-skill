# Work the signal queue

Signals are the strategist's **discovery queue** — non-obvious, audience-centered insights about this org's customers. This workflow is how you read them, decide what's worth acting on, and turn the good ones into campaigns. Full field/flag reference: [signals.md](../reference/commands/signals.md).

## The loop

```
hermes signals list                    # read the open queue, strongest first
hermes signals show <id>               # inspect a promising one in full
# decide: act on it, or dismiss it
#   → act:    author a trigger for that audience (see below)
#   → drop:   hermes signals dismiss <id>
hermes signals review <id>             # mark the ones you acted on / evaluated
```

Only run a fresh pass when the queue is stale or empty:

```
hermes signals discover                # expensive; capped at once per 2 days/org
```

## 1. Read before you discover

`hermes signals list` is free and instant. `hermes signals discover` spins up the strategist agent and is **rate-limited to once every 48 hours per org** — a second attempt inside that window returns `CONFLICT`. So always read the existing queue first; only discover when it's stale or empty and you need new ground.

Signals are ranked by `surprise` (non-obviousness) by default. `--sort recent` orders by when each was last re-observed. `observation_count` tells you how established a signal is — a high count means the org has seen it many times, so it's less fresh.

## 2. Decide: act or dismiss

A signal is a **lead, not a command**. For each one worth your attention, decide:

- **Act** — it points at an audience worth a campaign. Go to step 3.
- **Dismiss** — it's not actionable, already handled, or not worth it: `hermes signals dismiss <id>`.

Mark the ones you engage with: `hermes signals review <id>` keeps a signal in the open queue but records that you've dealt with it, so the queue reflects what's actually outstanding.

## 3. Act on a signal → author a trigger

There is **no automatic conversion**. A signal deliberately carries no detection query — it describes *who* and *why*, not *how to find them*. To act, you author a trigger the normal way, using the signal's `who` / `insight` / `so_what` as the brief:

```
hermes connections list                        # pick the source the cohort lives in
hermes connections schema [<id>]               # read the dialect + columns
hermes connections query "<candidate>" [<id>]  # iterate until the cohort matches the signal's `who`
hermes triggers create \
  --detection-query "<validated query>" \
  --email-prompt "<brief drawn from the signal's insight + so_what>" \
  [--source <id>]
```

The signal tells you the *target* and the *angle*; you translate that into a query against the live source and a prompt that speaks to the behavior that defines the cohort. See [create-trigger.md](create-trigger.md) for the full trigger-authoring loop.

After the trigger exists, mark the signal: `hermes signals review <id>`.

## 4. Surface fresh signals (when warranted)

```
hermes signals discover
```

By default the CLI polls until the pass finishes and reports how many signals were surfaced, then read them with `hermes signals list`. Use `--no-wait` to get the `job_id` and return immediately. Remember the 48-hour cooldown: if you (or the daily cron) already ran discovery for this org today, this will `CONFLICT` — that's expected, just work the existing queue.

## Notes

- **Cheap read, expensive write.** Listing/triaging is free; discovery is the one rate-limited, costly operation here.
- **Triage keeps the queue honest.** Dismiss what you rule out, review what you act on — otherwise the open queue stops meaning "outstanding."
- **You are the judgment.** Hermes surfaces the signal; deciding whether (and how) to act on it is your call.
