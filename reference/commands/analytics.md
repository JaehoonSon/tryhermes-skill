# `hermes analytics`

Aggregate metrics for campaigns and the org as a whole.

## Subcommands

### `hermes analytics campaign <trigger-id> [--since <duration>]`

Per-trigger metrics:

```
Trigger: welcome-new-users
Window:  last 30d

Reached:        1,243
Sent:           1,201
Delivered:      1,189   (98.9%)
Opened:           512   (43.1%)
Clicked:           87   ( 7.3%)
Replied:           14   ( 1.2%)
Bounced:           12   ( 1.0%)
Unsubscribed:       4   ( 0.3%)
```

### `hermes analytics overview [--since <duration>]`

Org-wide aggregate across all triggers, plus a per-trigger breakdown table.

```
Org:    acme-marketing
Window: last 30d

Total reached:    8,412
Total sent:       8,201
Delivered rate:   98.7%
Open rate:        41.3%
Click rate:        6.9%
Reply rate:        1.1%

Top triggers by reach:
  welcome-new-users          1,243
  feedback-power-users         812
  reactivation-30d             604
  ...
```

## Flags shared by both

| Flag | Default | Description |
|---|---|---|
| `--since <duration>` | `30d` | Time window. Examples: `24h`, `7d`, `90d` |
| `--until <iso-date>` | now | End of window |
| `--json` | off | Machine-readable output |

## Notes

- **Reach** = unique users that entered the trigger's audience.
- **Open** and **click** rates are based on **delivered** emails, not sent.
- **Reply rate** counts inbound messages threaded to a campaign email.
- Metrics update as webhook events arrive; very recent windows (last few minutes) may undercount.
