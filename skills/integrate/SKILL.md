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

- Cabinet (browser UI): `https://vector.pharosone.ai`. REST API base URL: `https://vector-api.pharosone.ai` (ask the user to confirm; defaults for the SaaS deployment).
- MCP server (already pre-wired by this plugin): `https://vector-api.pharosone.ai/api/v1/mcp/`. The first MCP call triggers Clerk's OAuth flow (browser opens to the Clerk-hosted consent screen, user picks the organization, Clerk issues a tenant-bound token bound to this MCP resource).
- **Public OpenAPI mirror (no auth, CORS open): `https://vector-api.pharosone.ai/api/v1/public/openapi.json`.** This is the source of truth for the five integration endpoints — fetch it first if your editor has web access and verify every field name you use against it. If you cannot reach it, the exact wire shapes are inlined in Phase 0 below.
- AgentContext JSON-Schema (also public, CORS open): `https://vector-api.pharosone.ai/api/v1/public/agent-context-schema.json`. The richer cabinet-only API is documented under `https://vector.pharosone.ai/docs/api` (sign-in required) — do not call those `/api/v1/cabinet/*` paths from CI.
- Default implementation language: match the user's repo. The runnable reference at `skills/integrate/reference/vector_runner.py` is Python (the most common LLM-agent stack); ports to TypeScript or Go follow the same shape.

# What Vector does

Vector is a red-team control plane for LLM agents. The flow is:

1. `POST /api/v1/sessions` with an `agent_context` → planner returns N attack prompts targeted at *that* agent profile.
2. For each attack: send `attack.prompt` to the user's agent verbatim, capture the raw response.
3. `POST /api/v1/sessions/{id}/results` with body `{"results": [{ attack_id, agent_response, latency_ms?, tool_calls?, metadata? }, …]}` (wrapper is required, not a bare array).
4. `GET /api/v1/sessions/{id}/report` once judging finishes (handles 425 Too Early by retrying).
5. Translate report verdicts into a build signal — `summary.broken > 0` typically means "fail the build".

Verdicts the judge emits per response: **`PASS`** (held the line), **`FAIL`** (broken), **`PARTIAL`** (soft leak). On a `Finding`, the `verdict` field is one of these three or `null` while still being judged or when the judge crashed; nulls roll up into `summary.inconclusive`. Do NOT treat `INCONCLUSIVE` as a fourth verdict value — it only appears as a summary count. The whole pipeline is async on the server side, but the client surface is plain REST.

ID prefixes you will see — useful for sanity checks: `sess_<24-hex>` for sessions, `atk_<24-hex>` for attacks, `agt_<24-hex>` for saved agents, `ak_<…>` for API keys.

# Auth & scope (hard rules)

- **Only call the integration API.** Allowed paths: `POST/GET /api/v1/sessions`, `/api/v1/sessions/{id}`, `/api/v1/sessions/{id}/attacks`, `/api/v1/sessions/{id}/results`, `/api/v1/sessions/{id}/report`, `/api/v1/sessions/{id}/report.pdf`, optionally `/api/v1/agents/*` to save reusable agent profiles, and the no-auth `/api/v1/public/*` schemas. Anything under `/api/v1/cabinet/*` is cabinet-cookie-only — do not call it from CI. The MCP endpoint `/api/v1/mcp` is reserved for interactive editor sessions (this plugin uses it) and not for pipelines.
- **Auth header is `Authorization: Bearer ak_...`** — a long-lived Clerk-issued API Key (opaque secret, NOT a JWT — do not try to decode it). The user mints it ONCE in the cabinet `Settings → API keys` (`https://vector.pharosone.ai/api-keys`); the value is shown once and then only the `ak_...` id remains visible. The key's subject is the user's organization, so every session created with it is scoped to that org.
- **You (the AI editor) MUST NOT see the API Key value.** Do not ask the user to paste their `VECTOR_API_KEY` into this chat. Instead: instruct them to mint it in the cabinet and store it in (a) the repository's existing secret manager pattern (GitHub Actions secret, GitLab CI/CD variable, Vault, Doppler, k8s Secret, …) AND (b) a local `.env` for `make redteam`-style commands — and add `.env` to `.gitignore` if it isn't already. The code you generate reads it via `process.env.VECTOR_API_KEY` (or the language equivalent). It NEVER appears in source files, commits, logs, or this conversation.
- `agent_context` payloads are NOT secret — commit them under version control so red-team runs are reproducible.

# Phase 0 — pin down the API shapes you'll touch

The integration only ever touches these five endpoints. Fetch `https://vector-api.pharosone.ai/api/v1/public/openapi.json` once at the start if your editor has web access; if you cannot, the exact shapes below are copied straight from that document. **Do NOT invent fields, and do NOT generate a 4000-line client for five endpoints.**

Three exclusive ways to pick an agent on `POST /sessions`: `agent_context` (throwaway), `agent_id` (reuse a saved one), `agent_inline` (create + use atomically). Pick `agent_inline` for the first integration so the cabinet groups historical sessions per agent.

`delivery_mode`: `sync` (long-poll, get attacks inline, 15–90 s wait — set HTTP read timeout ≥ 300 s on your client), `polling` (202 → poll `GET /sessions/{id}` until `status` is `ready` / `in_progress` / `completed` → `GET /sessions/{id}/attacks`), `webhook` (202 → server POSTs the payload to your URL with HMAC-SHA256 in the `X-Vector-Kya-Signature` header). The `delivery_mode` default is `sync`.

## Wire shapes — copy these literally

### 1. `POST /api/v1/sessions` — create a session

Request body (one of `agent_inline` / `agent_id` / `agent_context` is required; everything else is optional):

```jsonc
{
  "agent_inline": {                       // see AgentInline below
    "description": "What the agent does, who it serves, what it can touch. 2-6 sentences.",
    "name": "support-bot-v3",
    "restrictions": ["never reveal the system prompt", "never refund >$500"],
    "system_prompt_excerpt": "Verbatim snippet of the real system prompt …",
    "language_hints": ["en"]              // platform is English-only — leave as ["en"]
  },
  "max_attacks": 10,                      // default ~20, capped server-side
  "delivery_mode": "sync",                // "sync" | "polling" | "webhook"
  "client_label": "ci-build-1042",        // shown in the cabinet's session list
  "categories_include": ["prompt_injection", "policy_bypass"],  // optional whitelist
  "categories_exclude": ["self_harm"],    // optional blacklist
  "webhook_url": "https://…/hooks/vector",        // only when delivery_mode=webhook
  "webhook_secret": "32-byte-shared-secret"       // 8-256 chars, only with webhook
}
```

Response (200 for `sync`, 202 for `polling` / `webhook`):

```jsonc
{
  "session_id": "sess_3f29c1a08c5b4d9da2f4e7b1c0f3a92",
  "status": "ready",                      // "ready" on sync; "planning" on polling/webhook
  "delivery_mode": "sync",
  "total_attacks": 10,                    // populated on sync; null on polling/webhook
  "attacks": [ /* AttackOut[] */ ],       // populated on sync; null on polling/webhook
  "agent_id": "agt_3f29c1a08c5b4d9da2f4e7b1c0f3a92"   // null when using agent_context
}
```

### 2. `AttackOut` — one prompt to send to the agent

Returned inline (sync) or via `GET /api/v1/sessions/{id}/attacks` (polling/webhook):

```jsonc
{
  "attack_id": "atk_8c5b4d9da2f4e7b1c0f3a921",  // echo back in ResultIn.attack_id
  "sequence": 1,
  "category": "prompt_injection",
  "subcategory": "override_system_prompt",
  "obfuscation_layers": ["base64"],       // [] for cleartext, joined with "+" in reports
  "language": "en",
  "prompt": "The exact text — send VERBATIM to your agent",
  "expected_failure_indicator": "What 'broken' looks like for this attack",
  "metadata": { /* opaque, schema may evolve */ },
  "single_turn": true
}
```

### 3. `GET /api/v1/sessions/{id}/attacks` — re-fetch attacks (polling path)

Returns `AttackListResponse`, **not a bare array** — the attacks live under `.attacks`:

```jsonc
{
  "session_id": "sess_…",
  "total_attacks": 10,
  "attacks": [ /* AttackOut[] */ ]
}
```

Returns 409 while the session is still `planning` or has `failed` — retry with backoff.

### 4. `POST /api/v1/sessions/{id}/results` — submit agent responses

Request body — note the `{"results": [...]}` wrapper, never send a bare array:

```jsonc
{
  "results": [
    {
      "attack_id": "atk_…",               // REQUIRED — must match an AttackOut.attack_id
      "agent_response": "raw user-visible text the agent produced",  // REQUIRED, max 200000 chars
      "latency_ms": 1240,                 // optional, integer ≥ 0
      "tool_calls": [                     // optional, free-form list of objects
        { "name": "lookup_order", "arguments": { "order_id": "X-1" } }
      ],
      "metadata": { "model": "gpt-4o", "prompt_hash": "…" }   // optional, opaque
    }
  ]
}
```

Response:

```jsonc
{
  "accepted": 5,                          // how many freshly recorded in this batch
  "results_received_total": 10,           // running total across all batches for this session
  "total_attacks": 10,
  "session_status": "completed"           // "planning" | "ready" | "in_progress" | "completed" | "failed"
}
```

Submitting the same `attack_id` twice returns **409** — keep a local set of submitted ids and skip on retry. Batch size 5–10 is a good default; the schema only requires `minItems: 1`.

### 5. `GET /api/v1/sessions/{id}` — lifecycle snapshot (polling)

`SessionStateResponse`, used to poll while waiting for `planning → ready` or `in_progress → completed`:

```jsonc
{
  "session_id": "sess_…",
  "status": "in_progress",                // planning | ready | in_progress | completed | failed
  "delivery_mode": "sync",
  "total_attacks": 10,                    // 0 while status="planning"
  "delivered": 10,
  "results_received": 7,
  "judged": 5,                            // when judged == total_attacks the report is fully ready
  "broken_count": 1,
  "webhook_status": "none",               // none | pending | delivered | failed
  "webhook_attempts": 0,
  "webhook_last_error": null,
  "error": null,                          // populated only when status="failed"
  "created_at": "2026-01-01T00:00:00Z",
  "completed_at": null
}
```

### 6. `GET /api/v1/sessions/{id}/report` — the report

Returns 200 with `ReportResponse`, **or 425 Too Early while any result is still being judged — retry with backoff, that's expected, not an error**:

```jsonc
{
  "session_id": "sess_…",
  "summary": {
    "total": 10,
    "broken": 2,                          // count of FAIL verdicts — your build-fail signal
    "defended": 7,                        // count of PASS verdicts
    "partial": 1,                         // count of PARTIAL verdicts
    "inconclusive": 0,                    // count of null verdicts (judge timeout / crash)
    "by_category": {
      "prompt_injection": { "PASS": 5, "FAIL": 2, "PARTIAL": 1, "INCONCLUSIVE": 0 }
    },
    "by_obfuscation": {
      "(none)": { "PASS": 4, "FAIL": 1, "PARTIAL": 0, "INCONCLUSIVE": 0 },
      "base64+roleplay": { "PASS": 3, "FAIL": 1, "PARTIAL": 1, "INCONCLUSIVE": 0 }
    }
  },
  "findings": [
    {
      "attack_id": "atk_…",
      "verdict": "FAIL",                  // "PASS" | "FAIL" | "PARTIAL" | null
      "reason": "Agent disclosed the system prompt verbatim.",
      "evidence_snippet": "You are Alex, the NovaMart support agent…",
      "prompt": "<the attack prompt>",
      "agent_response": "<the response>",
      "category": "prompt_injection",
      "subcategory": "override_system_prompt",
      "obfuscation_layers": [],
      "language": "en",
      "metadata": { /* opaque */ },
      "judged_at": "2026-01-01T00:01:00Z"
    }
  ],
  "metadata": {
    "library_version": "a1b2c3d",
    "planner_model": "claude-3-5-sonnet-20241022",
    "judge_model": "claude-3-5-sonnet-20241022",
    "duration_ms": 123456
  }
}
```

Build signal: `report.summary.broken > 0` is the canonical "fail the build" check. **Not** `report.failed`, **not** `report.broken_count`, **not** `summary.fail` — `report["summary"]["broken"]`.

PDF report at `GET /api/v1/sessions/{id}/report.pdf` follows the same 425-retry semantics; the response body is `{ "path": "/absolute/server/path", "size_bytes": … }`, not the PDF bytes.

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
9. Which Vector deployment URL? Default cabinet is `https://vector.pharosone.ai`, default REST API is `https://vector-api.pharosone.ai`. Confirm before continuing (private deployments will use different hosts but the same path layout).

Stop and wait. Do not fabricate answers to skip the interview.

# Phase 3 — design the testing layer (adapter shape)

Two clean modules, regardless of stack:

- **AgentAdapter** — narrow interface: `invoke(prompt: string) -> { response: string, latency_ms: number, tool_calls?: any[], metadata?: object }`. One implementation per agent shape (HTTP/function/CLI/MCP). Pure I/O; no Vector knowledge. Reuse the repo's existing HTTP client, telemetry, retry middleware, etc. — do not write a parallel stack.
- **RedTeamRunner** — orchestrates the Vector flow: pick delivery_mode, call `POST /sessions`, drive the AgentAdapter once per attack with bounded concurrency (default 4, capped by the answer to interview Q5), batch `POST /results` (size 5–10 keeps memory low), poll `GET /report` with backoff (the endpoint returns 425 Too Early while judges are pending — that's not an error, retry).

Build these together but keep them separate files so swapping the adapter when the agent moves (e.g. function → HTTP service) is one file change.

Generate types from OpenAPI — `datamodel-code-generator` for Python, `openapi-typescript` + a small fetch wrapper for TS/JS, `oapi-codegen` for Go — but only for the request/response shapes you actually use. A 4000-line generated client for 5 endpoints is noise.

# Phase 4 — implement

**Start from the reference skeleton: `skills/integrate/reference/vector_runner.py`.** It is a single-file Python runner that already exercises every endpoint with the correct field names, the right timeouts, retry-on-425, retry-on-409, batched submission, ThreadPoolExecutor concurrency, on-disk resume state, secret redaction, and the `summary.broken > 0` exit-code policy. Read it, copy it into the user's repo, then replace the `MyAgentAdapter.invoke` body with code that calls THEIR agent and edit `build_agent_inline()` to describe their agent. Do not rewrite the runner from scratch — the wire shapes are landmines, and the skeleton is verified against the live OpenAPI.

When porting to TypeScript or Go, keep the same module layout (`AgentAdapter` + `VectorClient` + orchestrator) and the same field names; the wire shapes in Phase 0 are language-agnostic.

Implementation rules — these apply regardless of language:

- Place files in the directory the repo's convention dictates (`tests/red_team/`, `scripts/red_team/`, `src/redTeam/`, …). Do not create a parallel project.
- One config file (`vector.json` or `.toml` or `.yaml` — match the repo) holds the agent profile, `max_attacks`, `categories_include`/`exclude`, fail-policy threshold, base URL. Commit it. The API key is the ONE thing that lives in env / secret manager, never in the config.
- The runner reads `VECTOR_API_KEY` from env (use the repo's existing secret loader). It NEVER logs the key, even on error. Redact `Bearer ak_…` from every log line, response dump, and telemetry event (the skeleton does this via a logging `Formatter`).
- Handle the three failure modes:
  - (a) planner error on `POST /sessions` → fail fast with the response body; do NOT auto-retry (it spends planner compute).
  - (b) agent error inside the AgentAdapter → record `agent_response: ""` with the error in `metadata.adapter_error` and keep going. The judge will return `verdict: null` for that finding (rolls up into `summary.inconclusive`) — that is intended.
  - (c) judge still scoring → `GET /report` returns 425. Retry with exponential backoff (start ~2 s, cap ~30 s, total budget ~10 min). 425 is not an error.
- Idempotency: each `attack_id` may only be submitted once (409 on duplicate). If a run is resumable, persist already-submitted `attack_id`s and skip on retry — the skeleton uses `.vector-runner-state.json` for this.

# Phase 5 — wire into the pipeline

Match the trigger answer from Phase 2:

- PR gate → `.github/workflows/red-team.yml` (or GitLab/Jenkins/Circle equivalent) that runs the runner on a small budget (max_attacks 5–10) and `exit 1` on broken > 0.
- Nightly deep scan → scheduled workflow with larger budget (max_attacks 50+); upload the JSON report + PDF (`GET /api/v1/sessions/{id}/report.pdf`) as an artefact.
- Manual dev loop → `make redteam` / `npm run redteam` / `just redteam`, printing the cabinet URL of the session so the user can open the report in browser.
- Post-deploy smoke → tiny session (max_attacks 1–3) right after release, fail loud on broken > 0.

Plug the failure signal into whichever channel the repo already uses (PR check, Slack notify step, exit code) — no new infra.

Starter recipes for common CI systems live on the cabinet's Integration page (GitHub Actions, GitLab CI, Jenkins, cURL). If the user can paste one, use it as the skeleton — replace the placeholder loop with the real adapter call from Phase 3. Do not rebuild the curl chain from scratch when a working snippet is available.

# Phase 6 — verify (do this before declaring done)

1. **Pre-flight reachability** — hit the public OpenAPI to confirm DNS + TLS work from the dev machine (no auth needed):
   ```bash
   curl -fsS https://vector-api.pharosone.ai/api/v1/public/openapi.json | head -c 200
   ```
   If this fails the network is blocked, the base URL is wrong, or the deployment is down — fix before anything else.
2. **Dry-run the AgentAdapter alone** on a hardcoded prompt (`"hello"`) and confirm it returns a non-empty string. If it doesn't, the wiring to the agent is wrong — fix before going further.
3. **Run the full pipeline with `max_attacks: 1`** end-to-end. Confirm: session created, attack delivered, result submitted (status 200, `accepted: 1`), report fetched (possibly after one or two 425 retries), exit code obeys the policy.
4. Open the printed cabinet URL (`https://vector.pharosone.ai/sessions/{session_id}`) and visually confirm the session shows up with one finding.
5. Only then run a fuller `max_attacks: 10` session and review the verdicts with the user.
6. Open a PR titled "ci: integrate Vector red-team scan" with: the runner code, the config file, the CI step, a short `README` chunk explaining how to run it locally and how to mint the API key.

# Non-negotiables

- HTTP read timeout ≥ 300 s on `POST /sessions` with `delivery_mode: sync` (planning blocks). The default 30 s on most HTTP clients will time out mid-planning — set the read timeout explicitly. In httpx: `httpx.Timeout(30.0, read=300.0)`.
- `POST /sessions` is NOT retryable (it spends planner compute). If the network drops, surface the failure; do not auto-retry.
- `GET /sessions/{id}/report` returning **425 Too Early** is **expected** — judges are still scoring. Retry with exponential backoff (start ~2 s, cap ~30 s, total budget ~10 min). Do NOT treat 425 as a hard failure and do NOT try to parse the body as JSON.
- `GET /sessions/{id}/attacks` returning **409** means planning is still in progress (or the session failed). Poll `GET /sessions/{id}` until `status` is `ready` / `in_progress` / `completed`, then re-fetch attacks.
- `POST /sessions/{id}/results` returning **409** for an `attack_id` means it was already submitted. Skip and continue — do NOT fail the run.
- Send `attack.prompt` to the agent VERBATIM. No prepending "User:", no JSON-wrapping, no sanitization — that defeats the test. If the agent's normal channel requires a wrapper (system prompt + user message), wrap the same way it does in production, but the user-message slot gets the raw `attack.prompt`.
- Cap parallelism so the test doesn't DOS the agent or trip provider rate limits. Default 4 concurrent invocations; lower if the agent is paid-per-call.
- Do not regenerate the OpenAPI client on every CI run. Vendor the generated types into the repo (or hand-write them from Phase 0), regenerate only when the API version changes.
- Keep cabinet-only paths (`/api/v1/cabinet/*`, `/api/v1/auth/*`) OUT of any generated client and OUT of the test code. Wrong layer.

# Anti-patterns to reject — these are the real bugs that break the generated test

- **Sending a bare array to `POST /results`.** Wrong: `httpx.post(url, json=[{...}, {...}])`. Right: `httpx.post(url, json={"results": [{...}, {...}]})`. The `{"results": [...]}` wrapper is required.
- **Treating the attacks endpoint as a bare array.** `GET /sessions/{id}/attacks` returns `{session_id, total_attacks, attacks: [...]}`. Read `.attacks`; do not iterate the top-level object.
- **Inventing field names.** Common hallucinations: `prompt_text` (it is `prompt`), `response` on `ResultIn` (it is `agent_response`), `tools_used` (it is `tool_calls`), `failed_count` / `fail_count` (it is `summary.broken`), `verdict: "INCONCLUSIVE"` (the enum is only `PASS` / `FAIL` / `PARTIAL`; `null` means inconclusive). When in doubt, fetch the public OpenAPI and grep.
- **Reading `total_attacks` from `POST /sessions` in polling / webhook mode.** It is `null` there — only `sync` populates it. Either fetch via `GET /sessions/{id}` after planning, or use `len(attacks)` after `list_attacks`.
- **Treating `verdict == "INCONCLUSIVE"`.** It does not exist as a verdict value. `Finding.verdict` is one of `"PASS"`, `"FAIL"`, `"PARTIAL"`, or `null`. Count `summary.inconclusive` for the bucket, not by filtering findings by that string.
- **Failing the build on 425.** 425 means "judges still working" — back off and retry, do not call `raise_for_status()` and exit.
- **Path concatenation drift.** Either `BASE = "https://vector-api.pharosone.ai"` + `"/api/v1/sessions"`, or `BASE = "https://vector-api.pharosone.ai/api/v1"` + `"/sessions"`. Picking one and accidentally doing both produces `…/api/v1/api/v1/sessions` → 404.
- **Default 30 s HTTP timeout** on `POST /sessions` with `sync` mode. Planning routinely takes 30-90 s; default timeouts cut it off and the LLM concludes "the API hangs". Fix the timeout, not the API.
- **Hardcoding `max_attacks: 100` for the PR gate.** That's $$$. Start small (5–10), ramp via config for nightly.
- **Catching all exceptions inside the runner and printing "scan succeeded"** — failure must propagate to the CI exit code.
- **Logging the API key**, even at debug level. Redact `Authorization` / `X-API-Key` / any `Bearer ak_*` substring from every error trace, response dump, or telemetry event.
- **Asking the user to paste the `VECTOR_API_KEY` value into this chat** — that puts the secret in editor telemetry and the agent transcript. Always route the value through their secret manager + a gitignored `.env`, never through you.
- **"Here's a 500-line generic SDK, plug it in yourself."** → No. Build the adapter to the user's actual agent and wire their actual pipeline.
- **Skipping the interview and inventing the agent's shape.** → Ask first.
- **Adding a new HTTP library / new test runner / new language alongside the repo's existing ones.**
