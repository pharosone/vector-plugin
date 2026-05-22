# Vector context for Gemini CLI

This Gemini CLI session has access to Vector, a red-team scanning service for LLM agents published by Pharos One. The Vector MCP server is bridged in through `mcp-remote` and exposes tools that mirror the public REST API (`create_session`, `get_session`, `list_attacks`, `submit_results`, `wait_for_report`, `get_report`, plus the `agents.*` CRUD).

## When to use MCP tools vs skills

- **Use MCP tools** when the user wants to do something with Vector data right now from this session: start a scan, fetch the latest report, list attacks for a session, save an agent profile. These return JSON; you summarize.
- **Use a skill** when the user wants to change their codebase in response to Vector output: integrate scanning into the repo / CI (`integrate`), fix a single FAIL (`harden-from-finding`), batch-fix many FAILs (`batch-fix-findings`), or generate an AgentContext JSON for the cabinet (`create-agent-context`).

Skills in Gemini CLI are not invoked through slash-commands — they activate when the user's plain-language request matches the skill's description. So "integrate Vector into this repo" maps to the `integrate` skill; "help me fix these red-team findings" maps to `batch-fix-findings`.

## Defaults

- Cabinet (browser UI): `https://vector.pharosone.ai`.
- REST API base URL: `https://vector-api.pharosone.ai`.
- MCP endpoint: `https://vector-api.pharosone.ai/api/v1/mcp/` (trailing slash matters — the server 307-redirects the bare `/mcp`; overridable via the `VECTOR_MCP_URL` env var for private deployments).
- Auth split (important):
  - **MCP tools (this session)** use OAuth 2.1 via Clerk — browser flow on first call, tokens cached by `mcp-remote`. No key for you to handle.
  - **REST API (the CI runner that the `integrate` skill writes into the user's repo)** uses `Authorization: Bearer ak_...`, a long-lived Clerk API Key minted by the user in the cabinet `Settings → API keys` page. **Do NOT ask the user to paste the API key value into this chat** — the value belongs in their secret manager and a gitignored `.env`, never in your context window.

## Auth flow on first MCP call

1. The `mcp-remote` bridge sends the request without auth.
2. The server returns 401 with `WWW-Authenticate: Bearer ... resource_metadata=<url>`.
3. The bridge fetches the resource metadata, opens the Clerk-hosted authorization URL in the user's browser.
4. The user picks which organization to act as, approves; Clerk issues access + refresh tokens bound to `https://vector-api.pharosone.ai/api/v1/mcp` (canonical resource — no trailing slash in the RFC 8707 audience claim, even though the request URL has one). The bridge caches them locally; subsequent calls succeed silently until they expire (default access token TTL: 1 hour, refresh TTL: 14 days; rotation is automatic).

If a tool call fails with a clear authentication error, suggest the user re-run the same command — the bridge will refresh tokens automatically, or reopen the consent flow if the refresh token has expired.
