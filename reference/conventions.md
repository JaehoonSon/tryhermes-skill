# Conventions

These apply to every command in the Hermes CLI.

## Help and discoverability

Every command — at every level — supports `--help` (alias `-h`). Help is auto-generated from the command's definition and lists subcommands, arguments, flags, and descriptions.

```bash
hermes --help                    # top-level: all command families
hermes triggers --help           # family-level: all subcommands
hermes triggers create --help    # leaf: args, flags, examples
```

Unknown commands print a "did you mean?" suggestion (`hermes triger` → "Did you mean `triggers`?").

Agents that haven't read this skill can still discover the full CLI surface by recursively running `--help`. The surface returned by `--help` is the contract — if it shows a flag, that flag works.

## Output

- **Default:** human-readable table or summary block.
- **`--json`:** structured JSON, suitable for scripting and agent consumption. Available on every read command and most write commands.

Always prefer `--json` when piping into other commands or composing in agent flows.

## Field shapes

Every resource (`connection`, `domain`, `sender`, `trigger`, `draft`, `email`, `organization`) maps **1:1 to its database row**. The JSON object returned by any `show`, `list`, `create`, or `update` command contains **every column** on the row, using the column's exact `snake_case` name and Postgres type.

This means:

- **No renaming.** What's in the DB column `cooldown_hours` is in the JSON field `cooldown_hours`. Not `cooldownHours`, not `cooldown`. Agents and scripts can use a single canonical name end to end.
- **No omission.** All columns are returned, including system-managed ones (`id`, `org_id`, `created_at`, `updated_at`). If a column is nullable and unset, it appears as `null`, not missing.
- **Update flags map directly.** Every `--flag-name <value>` on an `update` command writes to the column `flag_name` (kebab → snake). The per-command pages list every mutable column and its flag. Immutable columns (`id`, `org_id`, `created_at`, `updated_at`, plus content fields managed by NL flows like `detection_query` and `email_prompt`) are noted explicitly.
- **Secrets are redacted in CLI output but keep their column name.** Encrypted columns (e.g. `postgres_url`, `firestore_credentials`) appear as `"<redacted>"` rather than the plaintext value. The field name is preserved so agents can still detect "is this set?" without needing the secret.

Each command reference page has a **Fields** table listing every column on the underlying row: column name, type, default, whether it's settable on `create`, whether it's settable on `update`, and (if so) the corresponding flag. Treat that table as the source of truth.

## Async operations

Operations that enqueue background jobs (trigger run, email send, schema introspection, domain verification) return:

```json
{ "job_id": "uuid", "status": "queued" }
```

By default, the CLI **polls until terminal state** (`succeeded` or `failed`) and prints progress. Pass `--no-wait` to return immediately with the job id and check status later via `hermes jobs show <id>`.

## Errors

All errors return with a non-zero exit code and a structured payload (visible with `--json`):

```json
{
  "error": {
    "code": "DOMAIN_NOT_VERIFIED",
    "message": "Cannot create sender on unverified domain.",
    "hint": "Run `hermes domains verify example.com` after setting DNS records."
  }
}
```

Always read `hint` — it points at the next concrete step. Common error codes are listed in each command's reference page.

## Org resolution

Every command resolves the target org in this order:

1. `--org <slug>` flag (explicit override)
2. `./.hermes/project.json` from the current working directory (created by `hermes link`)
3. Error: `NO_LINKED_PROJECT`, hint: run `hermes link`

## Auth resolution

1. `HERMES_API_KEY` environment variable
2. `~/.hermes/config.json` (created by `hermes auth login --key …`)
3. Error: `NOT_AUTHENTICATED`, hint: run `hermes auth login`

## Destructive operations

`delete`, `remove`, and `unlink` require an explicit `--confirm` flag. The CLI does **not** prompt interactively — agents must pass `--confirm` themselves, which forces a deliberate decision.

## Idempotency

- `create` commands return an error if a uniqueness constraint would be violated (e.g., duplicate domain). Use `update` for changes.
- `activate` / `pause` / `approve` / `reject` are idempotent on the target state — re-running them on an already-matching row is a no-op, not an error.
