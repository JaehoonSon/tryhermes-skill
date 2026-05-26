# `hermes link`

Link the current working directory to a Hermes organization. Once linked, every subsequent command operates on that org without needing `--org` flags.

The link is stored in `./.hermes/project.json`:

```json
{ "org_slug": "acme-marketing", "org_id": "uuid" }
```

The CLI reads it on each request and sends `X-Hermes-Org: <slug>` so the server knows which org to operate in. For org-scoped API keys (`hk_…`) the header is redundant but ignored; for account-level CLI sessions (`cli_…`) it's required.

By convention, `.hermes/` is added to `.gitignore` (different orgs per environment is the common pattern). Teams that want a single shared org can commit it instead.

## Subcommands

### `hermes link --org <slug>`

Writes the link file after verifying the slug is one the current user has access to. Errors if not.

```
hermes link --org acme-marketing
```

(There is no interactive picker — CLI agents and most humans want explicit slugs. Use `hermes orgs list` to see what's available.)

### `hermes unlink --confirm`

Removes `./.hermes/project.json`. Does not affect the org server-side.

### `hermes status`

Prints the linked org, the active auth source (API key vs CLI session) with a masked prefix, and the API URL. Useful first call when debugging.

```
$ hermes status
API URL:  https://api.hermes.dev
Auth:     cli_session (cli_live_a1b2c3d4…)
Linked:   acme-marketing (3f2c…-uuid)
```

## Auth interaction

| Auth | Linked? | Behavior |
|---|---|---|
| Not signed in | — | Every command errors with `NOT_AUTHENTICATED`. Run `hermes auth login`. |
| API key (`hk_…`) | — | Works without a link — the key is already org-scoped. The link is optional but helpful for cross-checking. |
| CLI session (`cli_…`) | No | Every scoped command errors with `VALIDATION_FAILED` ("CLI sessions require an org context"). Run `hermes link --org <slug>` (or pass `--org <slug>` on the call). |
| CLI session (`cli_…`) | Yes | Every command auto-targets the linked org. |

## Common errors

| Code | Meaning | Hint |
|---|---|---|
| `VALIDATION_FAILED` | CLI session has no org context | Run `hermes link --org <slug>` or pass `--org <slug>` |
| `FORBIDDEN` | Slug doesn't exist or you're not a member | Run `hermes orgs list` to see what's available |
