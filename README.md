<div align="center">

# Vector

**Red-team scanning for LLM agents — wired into your editor.**

One plugin. Four AI editors. Continuous adversarial testing for the agents you ship.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](./LICENSE)
[![Plugin Version](https://img.shields.io/badge/version-0.1.2-7C3AED)](https://github.com/pharosone/vector-plugin/releases/tag/v0.1.2)
[![Claude Code](https://img.shields.io/badge/Claude_Code-supported-D97757)](#claude-code)
[![Cursor](https://img.shields.io/badge/Cursor-supported-000000)](#cursor)
[![Codex](https://img.shields.io/badge/Codex-supported-10A37F)](#codex)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-supported-4285F4)](#gemini-cli)
[![MCP](https://img.shields.io/badge/MCP-OAuth_2.1-7C3AED)](https://modelcontextprotocol.io)

[Install](#install) · [What you get](#what-you-get) · [How auth works](#how-auth-works) · [Self-hosted](#self-hosted--private-deployments) · [pharosone.ai](https://pharosone.ai)

</div>

---

## What is this

[Vector](https://pharosone.ai) is a red-team control plane for LLM agents, operated by [Pharos One](https://pharosone.ai). It plans targeted attacks against your specific agent, judges responses with an independent LLM, and produces actionable findings — `PASS` / `FAIL` / `PARTIAL` per attack, scored, categorized, mapped to AIUC-1 domains.

This plugin gives your AI editor first-class access to the Vector control plane and ships four skills that turn findings into pull requests:

```
your AI editor ─▶ Vector plugin ─▶ MCP (OAuth) ─▶ Vector SaaS ─▶ planner + judge
                                                                       │
                ◀──── findings ◀── report ◀────────────────────────────┘
                       │
                       ▼
              skills propose: adapter, CI step, narrow fix, regression test
```

No API keys to copy for in-editor use. The MCP server authorizes via OAuth 2.1 on first call — your browser opens to Clerk's consent screen, you approve, tokens are cached locally by the editor. (A long-lived API key is only needed for CI/CD — see [How auth works](#how-auth-works) below.)

---

## Install

### Claude Code

```bash
/plugin marketplace add pharosone/vector-plugin
/plugin install vector@vector
```

Then `/mcp` → pick `plugin:vector:vector` → approve in browser.

### Cursor

1. Cursor Settings (`⌘⇧J`) → **Plugins** → **Add from URL**
2. Paste `https://github.com/pharosone/vector-plugin`

> A Cursor Marketplace listing is in review. Until approval, use **Add from URL**.

### Codex

```bash
codex plugin marketplace add pharosone/vector-plugin
```

Inside Codex: `/plugins` → select **vector** → **Install**.

### Gemini CLI

```bash
gemini extensions install https://github.com/pharosone/vector-plugin
```

Gemini doesn't speak HTTP MCP natively, so the extension bridges through `mcp-remote@latest` (npx auto-installs the bridge on first use). Same OAuth flow.

---

## What you get

After install: **four skills** + **one MCP server**.

### Skills

| Skill | When to use | What it produces |
| --- | --- | --- |
| **`integrate`** | First-time setup | An `AgentAdapter` that calls your LLM agent, a `RedTeamRunner` driving the Vector REST API, a CI workflow, and a PR you can review |
| **`harden-from-finding`** | After a single `FAIL` | A targeted edit to your system prompt or tool definition + a regression test that replays the exact attack |
| **`batch-fix-findings`** | After a full session with several `FAIL`s | Findings grouped by **root cause**, one minimal patch per group, one regression test per group — not N fixes for N findings |
| **`create-agent-context`** | Creating a saved agent profile | Slug + 5-field `AgentContext` JSON ready to paste into the cabinet's **New agent** form |

### How to invoke a skill

| Editor | Invocation |
| --- | --- |
| Claude Code | `/vector:integrate`, `/vector:harden-from-finding`, etc. |
| Cursor / Codex | `/integrate`, `/harden-from-finding`, etc. |
| Gemini CLI | Plain language — *"integrate Vector into this repo"* matches the `integrate` skill |

In all four clients the skill descriptions are matched against your request automatically — you usually don't need to remember the names.

### MCP server

Pre-wired entry pointing at `https://vector-api.pharosone.ai/api/v1/mcp/`. After install, your editor's MCP panel will list `vector` with tools that mirror the SDK 1:1:

| Tool | What it does |
| --- | --- |
| `create_session` | Spin up a new red-team session (planner picks attacks for your agent) |
| `get_session` | State + progress of a session |
| `list_attacks` | Pull planned attack prompts to send to your agent |
| `submit_results` | Submit your agent's responses for judging |
| `wait_for_report` | Long-poll until the report is ready |
| `get_report` | Verdicts, summary, AIUC coverage, findings |
| `agents.*` | CRUD over saved agent profiles |

---

## How auth works

There are **two independent auth surfaces**, used by two different actors:

| Surface | Who calls it | Auth | Where you set it |
| --- | --- | --- | --- |
| **MCP tools** (this plugin) | Your AI editor in an interactive session | OAuth 2.1 (Clerk) — browser flow on first call | Pre-wired here — nothing to copy. You'll click once in a browser tab the editor opens. |
| **REST API** (`/api/v1/sessions`, `/api/v1/agents`, …) | CI runner (nightly cron, PR gate, post-deploy smoke) | `Authorization: Bearer ak_...` — long-lived Clerk API Key | Mint once in cabinet → **API keys** (`https://vector.pharosone.ai/api-keys`), store as a CI secret + local `.env`. The `integrate` skill writes code that reads it from `VECTOR_API_KEY`. |
| **Cabinet** (browser UI) | Human in a browser | Clerk session cookie | `https://vector.pharosone.ai` — sign in with your org |

Both auth surfaces speak to the same backend (`https://vector-api.pharosone.ai`) and see the same data, just through different doors. The plugin's skills prefer MCP whenever they read sessions or findings — no secret handling at all. The CI integration that `integrate` builds for your repo is the only place a long-lived key is needed, because CI runs unattended without a browser.

### First MCP call (OAuth 2.1, PKCE-S256, RFC 8707)

```
1. plugin  ─▶ POST /api/v1/mcp                 (no auth)
2. server  ─▶ 401  WWW-Authenticate: Bearer ... resource_metadata=<url>
3. plugin  ─▶ fetches resource metadata, opens Clerk consent URL in your browser
4. you     ─▶ pick the org you want to act as, approve
5. Clerk   ─▶ access_token (TTL 1h) + refresh_token (TTL 14d), bound to vector-api.pharosone.ai/api/v1/mcp
6. plugin  ─▶ caches tokens locally, retries the call — succeeds
```

Tokens are scoped to the user + organization you picked, and audience-bound to the Vector MCP endpoint. They cannot be replayed against any other API. Tokens are stored by your editor in its standard credential cache (`~/.cursor/...`, `~/.codex/...`, Claude Code's keychain entry, etc.) — never by this plugin.

### About the API Key (`ak_...`)

This is the **only** secret the integration touches, and the AI editor never sees its value:

- **Mint:** open `https://vector.pharosone.ai/api-keys` → New → copy the `ak_...` value once (the cabinet shows it exactly once).
- **Store:** paste it into your CI secret manager (GitHub Actions secret `VECTOR_API_KEY`, GitLab CI/CD variable, Vault, Doppler, k8s Secret, …) AND into a local `.env` for ad-hoc runs. Make sure `.env` is gitignored.
- **Use:** the `integrate` skill generates code that reads `process.env.VECTOR_API_KEY` (or the language equivalent) and sends it as `Authorization: Bearer ak_...`. The key never enters source files, never enters git history, never enters the AI editor's chat transcript.
- **Revoke:** same cabinet page — Revoke. Effective immediately; no rebuild needed.

API Keys are opaque (`ak_...`), not JWTs — do not try to decode them. Verification is a Clerk roundtrip per request; for typical CI rates (a handful of red-team runs per day) the latency is negligible.

---

## Self-hosted / private deployments

Set the `VECTOR_MCP_URL` environment variable before launching your editor:

```bash
export VECTOR_MCP_URL="https://your-vector-host.example.com/api/v1/mcp"
```

- **Cursor / Claude Code / Codex** — env var is expanded at runtime inside the manifest's `url` field.
- **Gemini CLI** — env var is passed directly to the `mcp-remote` bridge.

If your editor doesn't expand env vars in JSON manifests, edit `.mcp.json` locally to hardcode the URL.

> **Audience-binding contract (RFC 8707).** OAuth tokens are bound to the MCP `resource` URL. If you switch hosts, previously-issued tokens fail with `invalid_audience` — sign in again to re-mint.

---

## Repository structure

```
vector-plugin/
├── .claude-plugin/           Claude Code manifest + marketplace catalog
├── .cursor-plugin/           Cursor manifest
├── .codex-plugin/            Codex manifest (with interface metadata)
├── .mcp.json                 MCP server config (Claude Code + Cursor)
├── mcp.json                  MCP server config (Codex)
├── gemini-extension.json     Gemini CLI manifest (mcp-remote bridge)
├── GEMINI.md                 Gemini CLI context file
├── skills/                   Shared by all four clients
│   ├── integrate/
│   ├── harden-from-finding/
│   ├── batch-fix-findings/
│   └── create-agent-context/
├── assets/
│   └── logo.svg
├── LICENSE
└── README.md                 you are here
```

Skill bodies are **plain markdown** with YAML frontmatter and zero client-specific code — adding support for a fifth editor would mean adding one more manifest file, not rewriting the skills.

---

## Versioning

- Version is declared in **five** manifests in lockstep (`.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, `.cursor-plugin/plugin.json`, `.codex-plugin/plugin.json`, `gemini-extension.json`).
- Every release is tagged `vX.Y.Z` and published via [GitHub Releases](https://github.com/pharosone/vector-plugin/releases).
- Claude Code, Codex, and Gemini CLI pull `main` by default; pinning to a tag requires the user to install from a specific ref (`@vX.Y.Z`).

---

## Contributing

Bug reports and PRs welcome — open an [issue](https://github.com/pharosone/vector-plugin/issues) or send a pull request. For substantial changes, please open an issue first to discuss what you'd like to change.

---

<div align="center">

Made by [Pharos One](https://pharosone.ai). MIT licensed — see [LICENSE](./LICENSE).

</div>
