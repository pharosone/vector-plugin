# Vector AI plugin

Official Vector plugin for AI editors, published by [Pharos One](https://pharosone.ai). Adds four red-team scanning skills (`integrate`, `harden-from-finding`, `batch-fix-findings`, `create-agent-context`) and pre-wires the Vector MCP server so your editor can drive the SaaS directly.

Works in Claude Code, Cursor, OpenAI Codex, and Gemini CLI from a single repo.

## Installation

### Claude Code

```bash
/plugin marketplace add pharosone/vector-plugin
/plugin install vector@vector
```

Then run `/mcp`, pick `plugin:vector:vector`, and follow the browser prompt to authorize OAuth.

### Cursor

Install from the [Cursor Marketplace](https://cursor.com/marketplace) (once approved) or add manually:

1. Cursor Settings (Cmd+Shift+J) → Plugins → Add from URL
2. Paste `https://github.com/pharosone/vector-plugin`

The MCP server authorizes via OAuth on the first tool call — your default browser opens for the consent screen.

### Codex

```bash
codex plugin marketplace add pharosone/vector-plugin
```

Inside Codex run `/plugins`, select `vector`, install. OAuth runs on the first MCP call.

### Gemini CLI

```bash
gemini extensions install https://github.com/pharosone/vector-plugin
```

Gemini does not speak HTTP MCP natively, so the extension bridges through `mcp-remote@latest` (npx auto-installs the bridge on first use). Authorization is the same OAuth flow.

## What you get

After install, you get four skills and one MCP server.

### Skills

| Skill | When to use |
| --- | --- |
| `integrate` | First-time setup. Reads your repo, interviews you about how your LLM agent runs, builds an `AgentAdapter` + `RedTeamRunner`, wires a CI workflow, opens a PR. |
| `harden-from-finding` | After a single failed attack. Reads your system prompt, proposes a narrow fix for one `FAIL` finding, adds a regression test that replays the attack. |
| `batch-fix-findings` | After a full session with several `FAIL`s. Groups findings by root cause (one fix often closes multiple), proposes minimal edits per group, adds one regression test per group. |
| `create-agent-context` | Creating a saved agent profile. Interviews you and emits the `AgentContext` JSON to paste into the cabinet's New Agent form. |

Skill invocation is namespace-dependent:

- Claude Code: `/vector:integrate`, `/vector:harden-from-finding`, etc.
- Cursor / Codex: `/integrate`, `/harden-from-finding`, etc.
- Gemini CLI: invoke by plain language matching the skill description ("integrate Vector into this repo" → activates `integrate`).

In all four clients the skill descriptions are matched against your request automatically — you usually don't need to remember the names.

### MCP server

The plugin ships a pre-wired MCP entry that points at the SaaS endpoint (`https://mcp.pharosone.ai/mcp`). After install your editor's MCP panel will list `vector` with tools that mirror the SDK (`create_session`, `get_session`, `list_attacks`, `submit_results`, `wait_for_report`, etc. — 1:1 with the REST API).

Authentication is OAuth 2.1 — browser flow on first call, no API keys to copy.

## Self-hosted / private deployment

Set the `VECTOR_MCP_URL` environment variable to point at your deployment:

```bash
export VECTOR_MCP_URL="https://mcp.your-vector-host.com/mcp"
```

- In Cursor, Claude Code, and Codex the env var is expanded at runtime inside the manifest's `url` field.
- In Gemini CLI it's passed explicitly to the `mcp-remote` bridge — same env var name.

## Auth — what each surface uses

| Surface | Auth | Where |
| --- | --- | --- |
| MCP tools (this plugin) | OAuth 2.1 / browser flow | Pre-wired here |
| REST API (`/api/v1/sessions`, `/api/v1/agents`, …) | `X-API-Key` header | Mint a key in the cabinet (`/api-keys`) — used by the `integrate` skill when wiring CI |
| Cabinet (browser) | Session cookie | `/login` page |

The MCP server and the REST API are two doors to the same data. The plugin's skills will prefer MCP when they need to read sessions / findings (no key required); the CI integration produced by the `integrate` skill uses the REST API (key required, stored as a CI secret).

## Distribution

This plugin lives in a single repo (`pharosone/vector-plugin`) and is consumed by:

- `.cursor-plugin/plugin.json` → Cursor
- `.claude-plugin/plugin.json` + `.claude-plugin/marketplace.json` → Claude Code
- `.codex-plugin/plugin.json` → Codex
- `gemini-extension.json` + `GEMINI.md` → Gemini CLI

Skills live in `skills/<name>/SKILL.md` and are shared by all four clients without modification.

## License

MIT — see [LICENSE](./LICENSE).
