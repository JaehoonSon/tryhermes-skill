# `hermes auth`

Manage CLI authentication. Two parallel paths:

| Token type | Scope | Best for |
|---|---|---|
| **CLI session** (`cli_…`) | **Account-level** — acts on any org you're a member of. Org context per request comes from `./.hermes/project.json` (via [`hermes link`](link.md)) or the `--org` flag. | Humans driving the CLI interactively; LLM agents iterating on org setup. |
| **API key** (`hk_…`) | **Org-scoped** — bound to one org at mint time. | CI, scripted automation, per-environment isolation. |

Both flow through the same server middleware; routes never care which kind of token you used.

## Subcommands

### `hermes auth login`

Default: starts the **interactive device-code flow**. The CLI prints a URL and a short code; you open the URL in any browser, sign in (if needed), and click Authorize. The CLI polls in the background and saves the session token when approved.

```
$ hermes auth login

To authorize this CLI, open:
  https://app.hermes.dev/cli/authorize?code=ABCD-EFGH

Or visit https://app.hermes.dev/cli/authorize and enter the code:
  ABCD-EFGH

Waiting for approval (expires in 10 min)...
Authorized. Session saved.
```

The session is stored in `~/.hermes/config.json` (mode 0600). It survives shell sessions. `--no-wait` is not honored here — device flow is interactive by design.

### `hermes auth login --key <api-key>`

Bypasses the device flow. Validates the key against the API and stores it. Used for CI / scripted environments where there's no browser.

```
hermes auth login --key hk_live_abc123...
```

API keys are created in the Hermes dashboard (**Settings → API Keys**). Each key is scoped to one org (chosen at mint time), shows the full secret only once, and can be revoked at any time. For non-CI agent flows, prefer the device flow — the session token can act on any org without re-minting.

The two auth paths are **mutually exclusive in storage** — re-running `auth login` with `--key` replaces an existing session, and vice versa. Use `hermes status` to check what's active.

### `hermes auth whoami`

Prints the active user, the resolved org context (from the linked project or `--org`), and the auth source. Useful first call when debugging permissions.

For CLI sessions, `whoami` fails with a clear error if there's no linked project — that's expected. Run `hermes orgs list` to see what's available, then `hermes link --org <slug>`.

### `hermes auth logout`

Clears the stored credentials (whichever kind was set). Does not revoke server-side — go to the dashboard to revoke a session/key globally.

## Common errors

| Code | Meaning | Hint |
|---|---|---|
| `INVALID_API_KEY` | Token (API key or CLI session) not recognized, revoked, or expired | Run `hermes auth login` to re-authenticate |
| `NOT_AUTHENTICATED` | No token found in env or config | Run `hermes auth login` (interactive) or `hermes auth login --key …` (CI) |
| `VALIDATION_FAILED` | CLI session has no org context | Run [`hermes link --org <slug>`](link.md) or pass `--org <slug>` on the command |
| `FORBIDDEN` | You're not a member of the requested org | `hermes orgs list` to see your orgs |
