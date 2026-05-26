# Set up a sending domain and sender

You cannot send email until the org has at least one **verified domain** plus one **sender** on it. The middle step requires DNS records to be set at the domain's registrar — Hermes itself does not modify DNS.

## The flow

```
hermes domains add example.com         # Hermes registers domain, returns DNS records
↓
   *** DNS records get set at the registrar ***
↓
hermes domains verify example.com      # poll until status: verified
↓
hermes senders create --email hello@example.com --name "Acme Team"
```

## 1. Add the domain

```
hermes domains add example.com
```

Output is a table of DNS records (TXT for SPF/verification, CNAMEs for DKIM, possibly an MX). Example:

```
Domain: example.com
Status: pending

DNS records to set at your DNS provider:
  TYPE   HOST                          VALUE
  TXT    _amazonses.example.com        "abc123..."
  CNAME  s1._domainkey.example.com     s1.dkim.amazonses.com
  CNAME  s2._domainkey.example.com     s2.dkim.amazonses.com
  MX     example.com                   10 feedback-smtp.us-east-1.amazonses.com
```

Use `--json` to get the same data programmatically.

## 2. Set the DNS records

Each record from step 1 must exist at the domain's authoritative DNS provider **exactly as printed** — host, type, and value.

Then wait for propagation (typically 5–60 minutes) before running `verify`.

## 3. Verify

```
hermes domains verify example.com
```

Possible outcomes:
- `verified` — done; proceed to sender creation.
- `pending` — DNS hasn't propagated yet. Wait 5–10 minutes and retry, up to ~1 hour.
- `failed` — records don't match what was published. Re-run `hermes domains show example.com` and compare against what's actually in DNS (typos in TXT values are common; quoting and trailing dots vary by provider).

## 4. Create the sender

A sender is `<local-part>@<verified-domain>` plus a display name:

```
hermes senders create --email hello@example.com --name "Acme Team"
```

Recipients see this as `Acme Team <hello@example.com>`.

Common local-parts:
- `hello@`, `team@`, `hi@` — friendly, generic.
- `noreply@` — discouraged unless explicitly required; recipients can't reply.
- `<firstname>@` (e.g., `jaehoon@`) — feels personal, good for high-touch B2B campaigns.

Multiple senders per domain are fine — useful if the org runs different campaign tones (e.g., `support@` for transactional, `team@` for marketing).

## 5. Reference the sender on triggers

Every trigger needs a sender to actually send:

```
hermes triggers update <trigger-id> --sender <sender-id>
```

Without this, trigger activation may succeed but sends will fail with `NO_SENDER`.

## Multi-domain orgs

Some orgs send from multiple brands or sub-domains. Each domain needs its own verification:

```
hermes domains add brand-a.com
hermes domains add brand-b.com
```

Verify each independently, then create senders per brand. Triggers reference one sender, so per-trigger brand routing is automatic.
