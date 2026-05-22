---
name: create-agent-context
description: >-
  Generate an AgentContext JSON for Vector — interview the user about their
  LLM agent, produce a slug + 5-field JSON they paste into the cabinet's New
  Agent form. Use when creating a saved agent profile, or when the user asks
  "how do I describe my agent for Vector?".
metadata:
  author: Pharos One
---

# Goal

Generate an `AgentContext` payload for a Vector saved agent. The user will paste it into the New Agent page on the cabinet.

# Context

- Cabinet (default SaaS deployment): `https://vector.pharosone.ai`. REST API base: `https://vector-api.pharosone.ai`. If the user is on a different deployment, ask them for the URLs first.
- Live schema (CORS-open, no auth): `https://vector-api.pharosone.ai/api/v1/public/agent-context-schema.json`. Fetch this once at the start if your editor has web access — every field name used below must appear under `components.schemas.AgentContext.properties`. If it doesn't, the schema has been updated and you should trust the fetched schema, not this skill.

# The page has TWO inputs (you must produce TWO things)

1. A plain text input labelled "Name" above the JSON editor — that's the agent's slug (e.g. `support-bot-v3`). It is NOT inside the JSON.
2. The JSON editor — its content is **exactly** an `AgentContext` object (the 5-field schema below). No `{ name, description, agent_context: {...} }` wrapper around it.

# AgentContext — use ONLY these 5 fields, with these EXACT names

Required:

- `description` (string, 1–8000 chars) — Free-form prose: what the agent does, who it serves, what makes it risky, and (if relevant) which tools / data / external systems it can touch — written as prose, not a structured list. This is the single most important field; the planner reads it to choose attack categories. Aim for 3–6 sentences.

Optional, but each one sharpens a different attack class:

- `name` (string, ≤200) — human-readable display label shown in cabinet/reports (NOT the slug).
- `language_hints` (string[]) — **Always set to exactly `["en"]`.** This deployment supports only English-language agents.
- `restrictions` (string[]) — short imperative lines listing things the agent must NEVER do. Drives rule-evasion / policy-bypass attacks. 3–8 entries is a good range.
- `system_prompt_excerpt` (string, **target ≤500 chars; hard limit 4000**) — a short, tight snippet of the user's real system prompt: 2–4 sentences picked specifically because they contain policy verbs ("do not", "never", "always", "must", "refuse", "only", "before") or hard limits ("up to $500", "max 3 retries"). The planner uses these as attack anchors — a multi-paragraph dump adds noise, a focused snippet sharpens attacks. Paraphrase only if the real prompt is confidential.

The backend will silently accept a few more fields (`functions`, `privileges`, `communication_style`, `example_dialogue_start`, `example_dialogue_end`) but in practice they go unused — **do not generate them**. Keep the JSON to these 5 fields max.

# Common mistakes to AVOID

- DO NOT use `display_name` — the field is `name`.
- DO NOT use `languages` — the field is `language_hints`.
- DO NOT put anything other than `["en"]` in `language_hints`.
- DO NOT wrap the object in `{ name, description, agent_context: {...} }`. The JSON editor expects the AgentContext object directly.
- DO NOT include the agent slug inside the JSON — that's the separate "Name" input.
- DO NOT add `functions`, `privileges`, `communication_style`, or dialogue examples. They bloat the JSON without changing the run.
- DO NOT translate the description, restrictions, or system_prompt_excerpt into other languages — keep everything in English to match the platform's English-only policy.

# Tasks

1. Ask the user 3 short questions to fill the gaps:
   a. What's the agent — product, audience, and any tools / data it can touch (1–3 sentences)? Plus the slug for the "Name" field.
   b. What are the top 3–8 hard "never do" restrictions?
   c. Paste 2–4 sentences from the real system prompt — pick the ones that contain "do not" / "never" / "always" / "must" / hard numeric limits. Keep the total under ~500 characters (the form UI hints the same). Paraphrase only if the real prompt is confidential.

   If you have access to the user's repo, offer to extract candidate answers from the system prompt you can see (don't auto-apply — show them and ask to confirm).

2. Print exactly two blocks, in order:
   a) one line: `Name: <slug>` — to paste into the page's Name input.
   b) the `AgentContext` JSON in a fenced ```json``` block — to paste into the JSON editor.

3. (Optional, if your editor has web access) Fetch `https://vector-api.pharosone.ai/api/v1/public/agent-context-schema.json` and verify every field name you used appears under `components.schemas.AgentContext.properties`. If something doesn't match, the schema has been updated — trust the fetched schema.
