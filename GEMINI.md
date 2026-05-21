# Vector context for Gemini CLI

This Gemini CLI session has access to Vector, a red-team scanning service for LLM agents published by Pharos One. The Vector MCP server is bridged in through `mcp-remote` and exposes tools that mirror the public REST API (`create_session`, `get_session`, `list_attacks`, `submit_results`, `wait_for_report`, `get_report`, plus the `agents.*` CRUD).

## When to use MCP tools vs skills

- **Use MCP tools** when the user wants to do something with Vector data right now from this session: start a scan, fetch the latest report, list attacks for a session, save an agent profile. These return JSON; you summarize.
- **Use a skill** when the user wants to change their codebase in response to Vector output: integrate scanning into the repo / CI (`integrate`), fix a single FAIL (`harden-from-finding`), batch-fix many FAILs (`batch-fix-findings`), or generate an AgentContext JSON for the cabinet (`create-agent-context`).

Skills in Gemini CLI are not invoked through slash-commands — they activate when the user's plain-language request matches the skill's description. So "integrate Vector into this repo" maps to the `integrate` skill; "help me fix these red-team findings" maps to `batch-fix-findings`.

## Defaults

- Cabinet / REST API base URL: `https://app.pharosone.ai`.
- MCP endpoint: `https://mcp.pharosone.ai/mcp` (overridable via the `VECTOR_MCP_URL` env var).
- Auth: MCP uses OAuth (browser flow on first call). REST uses `X-API-Key` (the user mints it in the cabinet — only needed when the `integrate` skill wires CI).

## Auth flow on first MCP call

1. The bridge sends the request without auth.
2. The server returns 401 with `WWW-Authenticate: Bearer ... resource_metadata=<url>`.
3. The bridge fetches the resource metadata, opens the OAuth authorization URL in the user's browser.
4. After the user approves in the cabinet, tokens are stored locally; subsequent calls succeed silently until they expire (default access token TTL: 1 hour, refresh TTL: 14 days).

If a tool call fails with a clear authentication error, suggest the user re-run the same command — the bridge will refresh tokens automatically.
