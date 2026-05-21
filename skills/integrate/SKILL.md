---
name: integrate
description: >-
  Integrate Vector red-team scanning into the user's codebase — build an
  AgentAdapter that calls their LLM agent, a RedTeamRunner that drives the
  Vector REST API, and a CI workflow that fails the build on Broken
  findings. Use when setting up Vector for the first time, wiring it into
  CI, or adding red-team gates to a repo.
metadata:
  author: Pharos One
---

# Goal

Build a Vector red-team testing layer for the user's LLM agent and wire it into their existing pipeline. Do NOT just dump a generic HTTP client — design the adapter that calls their agent, pick the right delivery mode for their infrastructure, and integrate as a first-class step in whatever build/CI/cron system their repo already uses.

# Context

Vector is a SaaS red-team control plane for LLM agents, operated by Pharos One. Defaults for this plugin:

- Cabinet / REST API base URL: `https://app.pharosone.ai` (ask the user to confirm; this is the default SaaS deployment).
- MCP server (already pre-wired by this plugin): `https://mcp.pharosone.ai/mcp`. OAuth flow runs on first MCP call.
- OpenAPI (public integration surface — cookie-auth routes stripped): `https://app.pharosone.ai/api/v1/cabinet/openapi.json`.
- Default implementation language: match the user's repo. If they don't have a strong preference and the repo allows it, TypeScript is the default choice for new files.

# What Vector does

Vector is a red-team control plane for LLM agents. The flow is:

1. `POST /api/v1/sessions` with an `agent_context` → planner returns N attack prompts targeted at *that* agent profile.
2. For each attack: send `attack.prompt` to the user's agent verbatim, capture the raw response.
3. `POST /api/v1/sessions/{id}/results` with `[{ attack_id, agent_response, latency_ms?, tool_calls?, metadata? }, …]`.
4. `GET /api/v1/sessions/{id}/report` once judging finishes (handles 425 Too Early by retrying).
5. Translate report verdicts into a build signal — `summary.broken > 0` typically means "fail the build".

Verdicts the judge emits per response: `PASS` (held the line), `FAIL` (broken), `PARTIAL` (soft leak), `INCONCLUSIVE` (judge crashed / empty). The whole pipeline is async on the server side, but the client surface is plain REST.

# Auth & scope (hard rules)

- Only call the **integration** API: `POST/GET /api/v1/sessions`, `/api/v1/sessions/{id}`, `/api/v1/sessions/{id}/attacks`, `/api/v1/sessions/{id}/results`, `/api/v1/sessions/{id}/report`, optionally `/api/v1/agents` to save reusable agent profiles. Do NOT touch `/api/v1/cabinet/*`, `/api/v1/auth/*`, `/api/v1/oauth/*`, or `/api/v1/mcp` — those are for the browser cabinet and MCP clients, not pipelines.
- Auth header is `X-API-Key: <key>` (the user mints it in cabinet → API Keys). Never commit it; pull from an env var (`VECTOR_API_KEY`) using whatever secret pattern the repo already follows (`.env`, GitHub secrets, Vault, Doppler, k8s Secret, …).
- `agent_context` payloads are NOT secret — commit them under version control so red-team runs are reproducible.

# Phase 0 — read the API surface (do this first, once)

- Fetch `https://app.pharosone.ai/api/v1/cabinet/openapi.json` (or the deployment URL the user gave you). Skim `CreateSessionRequest`, `CreateSessionResponse`, `AttackOut`, `ResultIn`, `SubmitResultsRequest`, `SessionStateResponse`, `ReportResponse`, `ReportSummary`, `Finding`. These are the only shapes you need.
- Note the three exclusive ways to pick an agent on `POST /sessions`: `agent_context` (throwaway), `agent_id` (reuse a saved one), `agent_inline` (create + use atomically). Pick `agent_inline` for the first integration so the cabinet groups historical sessions per agent.
- Note `delivery_mode`: `sync` (long-poll, get attacks inline, 15–90 s wait — set HTTP read timeout ≥ 300 s on your client), `polling` (202 → poll `GET /sessions/{id}` until `status: ready` → `GET /sessions/{id}/attacks`), `webhook` (202 → server POSTs `WebhookPayload` to your URL with HMAC-SHA256 in `X-Vector-Kya-Signature`).

# Phase 1 — discover the user's repo (do not skip; do not guess)

Read what's actually there before suggesting anything:

1. Top-level layout, package manifests (`package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `Gemfile`, …) — pick the primary language.
2. Existing test runner (`pytest`, `vitest`, `go test`, `jest`, `rspec`, …) and CI config (`.github/workflows/`, `.gitlab-ci.yml`, `Jenkinsfile`, `circle.yml`, Makefile, npm scripts, `justfile`, `taskfile`, …).
3. **The agent under test.** It can have any shape — find which one this repo uses and read enough to call it without guessing:
   - HTTP endpoint (`/chat`, `/v1/messages`, FastAPI route, Express handler, Next.js Route Handler, …).
   - In-process function (`def chat(message: str) -> str`, `agent.run(messages)`, LangChain/LangGraph/LlamaIndex/AutoGen/CrewAI/Vercel AI SDK …).
   - CLI binary (`./mybot --prompt …`, a shell script).
   - Streaming / multi-turn protocol (SSE, WebSocket, MCP tool call) — flatten the final assistant text for the red-team adapter; preserve interim trace in `tool_calls`/`metadata` of `ResultIn`.
   - External SaaS (OpenAI Assistants, Anthropic API, Vertex, Azure OpenAI) that the repo wraps — call through whichever wrapper layer the repo already exposes.
4. Existing secret-handling convention (`.env.example`, `config/` module, `process.env.*` usages). Match it for `VECTOR_API_KEY`.
5. Where this CI normally signals failure (exit code, GitHub annotation, Slack webhook, Linear/Jira ticket). Plug into the same channel.

# Phase 2 — interview the user (ask, do not assume)

Print a SHORT numbered question block and **wait for answers** before writing files. Tailor the questions to what you just discovered; do not ask things the repo already answers. Typical fields:

1. Which entry point should I call as "the agent"? Confirm the file + function/route you found (or list candidates if it is ambiguous).
2. What does one round-trip cost (rough latency, $ / 1k calls) and what is a safe per-run budget? Drives `max_attacks` for the first runs (start at 5–10, ramp later).
3. What is the trigger? `on: pull_request` gate, nightly cron, on-demand `make redteam`, post-deploy smoke, manual GH Action — these need different `delivery_mode` choices.
4. Pass/fail policy: hard fail on any `broken`, threshold (`security_score ≥ 0.7`), warn-only with annotation, ratchet (must not regress vs. previous run)?
5. Concurrency limit on the agent under test (rate limits, queue, single-flight). Drives parallelism in the runner.
6. Where should results / report artefacts go (CI artefact, S3 bucket, posted as PR comment, attached to a Linear issue, Slack thread)?
7. Agent profile: a 3–6 sentence `description` of what the agent does + 3–8 `restrictions` (never-do rules) + a 2–4 sentence `system_prompt_excerpt`. Offer to extract these from the system prompt you can already see in the repo, then ask to confirm.
8. `delivery_mode` constraint: can the CI runner hold an HTTP request for 5 minutes? If no → `polling` or `webhook`. If webhook → is there a reachable HTTPS callback endpoint?
9. Which Vector deployment URL? Default is `https://app.pharosone.ai`. Confirm before continuing.

Stop and wait. Do not fabricate answers to skip the interview.

# Phase 3 — design the testing layer (adapter shape)

Two clean modules, regardless of stack:

- **AgentAdapter** — narrow interface: `invoke(prompt: string) -> { response: string, latency_ms: number, tool_calls?: any[], metadata?: object }`. One implementation per agent shape (HTTP/function/CLI/MCP). Pure I/O; no Vector knowledge. Reuse the repo's existing HTTP client, telemetry, retry middleware, etc. — do not write a parallel stack.
- **RedTeamRunner** — orchestrates the Vector flow: pick delivery_mode, call `POST /sessions`, drive the AgentAdapter once per attack with bounded concurrency (default 4, capped by the answer to interview Q5), batch `POST /results` (size 5–10 keeps memory low), poll `GET /report` with backoff (the endpoint returns 425 Too Early while judges are pending — that's not an error, retry).

Build these together but keep them separate files so swapping the adapter when the agent moves (e.g. function → HTTP service) is one file change.

Generate types from OpenAPI — `datamodel-code-generator` for Python, `openapi-typescript` + a small fetch wrapper for TS/JS, `oapi-codegen` for Go — but only for the request/response shapes you actually use. A 4000-line generated client for 5 endpoints is noise.

# Phase 4 — implement

- Place files in the directory the repo's convention dictates (`tests/red_team/`, `scripts/red_team/`, `src/redTeam/`, …). Do not create a parallel project.
- One config file (`vector.json` or `.toml` or `.yaml` — match the repo) holds `agent_context`, `max_attacks`, `categories_include`/`exclude`, fail-policy threshold, base URL. Commit it.
- The runner reads `VECTOR_API_KEY` from env (use the repo's existing secret loader). It NEVER logs the key, even on error. Redact `Authorization`/`X-API-Key` headers in any debug dump.
- Handle the three failure modes: (a) planner error on `POST /sessions` → fail fast with the response body; (b) agent error inside the AgentAdapter → record an empty `agent_response` with the error message in `metadata.error` and keep going (the judge will mark INCONCLUSIVE — that's intended); (c) judge timeout / inconclusive verdicts → exclude from pass/fail math, surface as a "retry later" warning.
- Idempotency: each `attack_id` may only be submitted once (409 on duplicate). If a run is resumable, persist already-submitted `attack_id`s and skip on retry.

# Phase 5 — wire into the pipeline

Match the trigger answer from Phase 2:

- PR gate → `.github/workflows/red-team.yml` (or GitLab/Jenkins/Circle equivalent) that runs the runner on a small budget (max_attacks 5–10) and `exit 1` on broken > 0.
- Nightly deep scan → scheduled workflow with larger budget (max_attacks 50+); upload the JSON report + PDF (`GET /api/v1/sessions/{id}/report.pdf`) as an artefact.
- Manual dev loop → `make redteam` / `npm run redteam` / `just redteam`, printing the cabinet URL of the session so the user can open the report in browser.
- Post-deploy smoke → tiny session (max_attacks 1–3) right after release, fail loud on broken > 0.

Plug the failure signal into whichever channel the repo already uses (PR check, Slack notify step, exit code) — no new infra.

Starter recipes for common CI systems live on the cabinet's Integration page (GitHub Actions, GitLab CI, Jenkins, cURL). If the user can paste one, use it as the skeleton — replace the placeholder loop with the real adapter call from Phase 3. Do not rebuild the curl chain from scratch when a working snippet is available.

# Phase 6 — verify (do this before declaring done)

1. Dry-run the AgentAdapter alone on a hardcoded prompt (`"hello"`) and confirm it returns a non-empty string. If it doesn't, the wiring to the agent is wrong — fix before going further.
2. Run the full pipeline with `max_attacks: 1` end-to-end. Confirm: session created, attack delivered, result submitted, report fetched, exit code obeys the policy.
3. Open the printed cabinet URL and visually confirm the session shows up with one finding.
4. Only then run a fuller `max_attacks: 10` session and review the verdicts with the user.
5. Open a PR titled "ci: integrate Vector red-team scan" with: the runner code, the config file, the CI step, a short `README` chunk explaining how to run it locally and how to mint the API key.

# Non-negotiables

- HTTP read timeout ≥ 300 s on `POST /sessions` with `delivery_mode: sync` (planning blocks).
- `POST /sessions` is NOT retryable (it spends planner compute). If the network drops, surface the failure; do not auto-retry.
- Send `attack.prompt` to the agent VERBATIM. No prepending "User:", no JSON-wrapping, no sanitization — that defeats the test. If the agent's normal channel requires a wrapper (system prompt + user message), wrap the same way it does in production, but the user-message slot gets the raw `attack.prompt`.
- Cap parallelism so the test doesn't DOS the agent or trip provider rate limits. Default 4 concurrent invocations; lower if the agent is paid-per-call.
- Do not regenerate the OpenAPI client on every CI run. Vendor the generated types into the repo, regenerate only when the API version changes.
- Keep cabinet-only paths (`/api/v1/cabinet/*`, `/api/v1/auth/*`) OUT of the generated client and OUT of the test code. Wrong layer.

# Anti-patterns to reject

- "Here's a 500-line generic SDK, plug it in yourself." → No. Build the adapter to the user's actual agent and wire their actual pipeline.
- Skipping the interview and inventing the agent's shape. → Ask first.
- Hardcoding `max_attacks: 100` for the PR gate. → That's $$$. Start small, ramp via config.
- Adding a new HTTP library / new test runner / new language alongside the repo's existing ones.
- Logging the API key, even at debug level.
- Catching all exceptions inside the runner and printing "scan succeeded" — failure must propagate.
