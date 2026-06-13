# Hosted MCP server — connecting agents to Hermes

Hermes exposes a hosted (remote) MCP server at **`https://tryhermes.dev/api/mcp`** (Streamable HTTP). It serves a typed tool catalog mirroring the v1 API surface — the same operations as [api-surface.md](api-surface.md), one named tool per endpoint (`list_triggers`, `create_connection`, `approve_draft`, …) — so any MCP-capable client can drive Hermes with nothing to install or self-host.

## Connecting

| Client | How |
|---|---|
| **claude.ai** | Settings → Connectors → **Add custom connector** → URL `https://tryhermes.dev/api/mcp`. OAuth flow: sign in to Hermes and approve — the grant covers your whole account (every org you belong to). |
| **Claude Code (OAuth)** | `claude mcp add --transport http hermes https://tryhermes.dev/api/mcp`, then run `/mcp` to complete the browser sign-in (same sign-in as claude.ai; the grant covers every org). |
| **Claude Code (API key)** | `claude mcp add --transport http hermes https://tryhermes.dev/api/mcp --header "Authorization: Bearer hk_…"` — key minted under Settings → API keys in the dashboard. |
| **Cursor / other header-capable clients** | Point at the URL and send `Authorization: Bearer hk_…` (e.g. `{ "url": "https://tryhermes.dev/api/mcp", "headers": { "Authorization": "Bearer hk_…" } }` in `mcp.json`). |

## Tokens

- **Account-level** — OAuth-minted tokens are CLI sessions covering every org on your account. Tools default to your only org, or take an `org: <slug>` argument when you belong to several. (`hk_` API keys remain org-pinned for CI/header use.)
- **Revocable** — OAuth connections are minted as CLI sessions named "Claude connector — <email>"; revoking the session cuts the client off immediately.
- **90-day expiry** — OAuth-minted tokens expire after 90 days; reconnect the connector to sign in again. (Dashboard-minted `hk_` keys don't expire unless revoked.)
