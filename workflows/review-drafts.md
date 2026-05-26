# Review and send drafts

When a trigger fires without `auto_send`, every detected user becomes a **draft** — an AI-personalized email waiting for human approval. This workflow covers the approve / reject flow.

## The queue

```
hermes drafts list                              # all `drafted` across the org
hermes drafts list --trigger <id>               # one campaign
hermes drafts list --trigger <id> --json        # for scripting
```

## Inspect before approving

```
hermes drafts show <draft-id>
```

Always check the body in agent-driven flows. AI-drafted copy is usually correct, but per-user personalization can occasionally hallucinate (wrong feature name, wrong tone for a high-value account). The cost of one bad email to a key user is high; the cost of inspection is low.

## Approve

Single:
```
hermes drafts approve <draft-id>
```

Bulk (for one trigger):
```
hermes drafts approve --all --trigger <trigger-id>
```

Bulk approve enqueues each draft as a separate send job. Failures don't block other sends.

## Reject — and what it means

```
hermes drafts reject <draft-id> --reason "tone too aggressive"
```

Rejection has an important side effect: **the trigger's claim on this user is released**, so the user is eligible for re-detection on the next trigger run. This is the right path when:
- The draft isn't quite right and you want Hermes to try again later (likely with refined trigger settings).
- You want this user to receive *some* email from this trigger, just not this one.

Rejection is **not** the right path when the trigger itself is wrong — refine or pause the trigger; don't reject drafts one by one.

## Recommended agent pattern

For any non-trivial campaign (more than ~20 drafts):

1. `hermes drafts list --trigger <id> --json` to enumerate.
2. Sample 3–5 with `hermes drafts show` and verify quality.
3. If samples look good: `hermes drafts approve --all --trigger <id>`.
4. If samples look off: `hermes triggers refine <id> "..."` or `hermes triggers pause <id>` and restart — don't try to fix the queue one draft at a time.

## After sending

```
hermes emails list --trigger <trigger-id>            # see what shipped
hermes analytics campaign <trigger-id> --since 24h   # see how it's performing
```

Replies arrive on the same email thread; inspect with `hermes emails show <id>`.
