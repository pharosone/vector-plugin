---
name: batch-fix-findings
description: >-
  Harden an LLM agent against a batch of FAIL findings from a single Vector
  session — group findings by root cause, propose minimal edits per group, add
  one regression test per group. Use after a full red-team session that
  produced multiple failures, or when the user says "fix all these findings".
metadata:
  author: Pharos One
---

# Goal

Harden the user's agent against N red-team findings from a single Vector session. The key insight: many findings share a root cause; one config change can close several at once. Do NOT propose N independent fixes.

# Context — gather the findings

You need: `session_id`, total Broken count, and for each finding: `attack_id`, `category / subcategory`, attacker prompt, agent's response, judge reasoning. Two ways to get them:

1. **MCP-first (preferred).** If the Vector MCP server is connected in this client (this plugin pre-wires it):
   - `get_session({ session_id })` to confirm the session is `completed`.
   - `wait_for_report({ session_id })` or `get_report({ session_id })` to fetch the report. The report's `findings` array is the source of truth — filter to verdict `FAIL` (and optionally `PARTIAL` if the user wants soft leaks too).
   Ask the user only for `session_id` and which verdicts to include.

2. **Manual paste (fallback).** Ask the user to:
   - Open the session in the cabinet (`/sessions/{session_id}` page).
   - Filter findings by `FAIL` verdict.
   - Paste the findings as a structured list (one block per finding with the six fields from `harden-from-finding`).

Per-finding caps for context window safety: prompt up to ~1500 chars, response up to ~2000 chars, judge reason up to ~800 chars. Truncate longer values with `…[truncated; N more chars in the full report]` and tell the user to open the cabinet for the full text.

# Tasks

1. Read the current system prompt and tool definitions of the agent in this repo.
2. **Group the failures by root cause.** Examples:
   - "weak refusal phrasing" (multiple bypass attempts succeeded because the refusal can be argued with)
   - "missing guardrail on tool X" (every attack that called tool X leaked something)
   - "system prompt leaked via tool description" (the tool description echoes part of the system prompt)
   - "PII handling in support flow" (multiple PII exfil attacks succeeded because there's no PII filter on the response)
   Multiple findings above often share a single fix. Print the groupings explicitly before proposing fixes.
3. For each group, propose concrete, minimal edits to the system prompt, tool definitions, or guardrail code. Quote the exact lines that should change and write the replacement inline. Explain WHY each edit addresses the root cause (so the user can sanity-check your reasoning).
4. Add a unit / integration test PER GROUP (not per finding) that:
   - Replays at least one representative attacker prompt from that group.
   - Asserts the safe behaviour.
   - Names the test after the group's root cause (e.g. `test_pii_filter_blocks_email_exfil`), not after a single `attack_id`.
5. **Do NOT over-correct.** Each fix must be the narrowest change that closes the specific gap. Blanket refusals harm normal traffic; flag any proposed change that risks normal-use regression and ask before applying.
6. After the changes, recommend a follow-up Vector scan (re-running `POST /api/v1/sessions` against the same `agent_context`, or "Re-run" in the cabinet, or the MCP `create_session` tool with the same context) so the user can confirm the fixes hold.

# Verification before declaring done

- Run every new test and confirm they all pass against the patched agent.
- Summarize: N findings grouped into M root causes, M fixes applied, M regression tests added.
- Print the cabinet URL for the original session so the user can compare verdicts side-by-side after the re-run.
