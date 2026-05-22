---
name: harden-from-finding
description: >-
  Harden an LLM agent against a specific FAIL verdict from Vector — read
  the system prompt, propose a narrow fix, add a regression test that replays
  the attack. Use when the user pastes a single failed finding from a session
  report or asks "how do I fix this red-team finding?".
metadata:
  author: Pharos One
---

# Goal

Harden the user's agent against a specific red-team finding from Vector.

# Context — gather the finding details

You need five fields about the failed finding: `session_id`, `attack_id`, `category / subcategory`, the attacker prompt, the agent's broken response, and the judge's reasoning. There are two ways to get them:

1. **MCP-first (preferred).** If the Vector MCP server is connected in this client (this plugin pre-wires it), use the MCP tools:
   - `get_session({ session_id })` → confirms the session exists and is `completed`.
   - `list_attacks({ session_id })` → finds the attack matching `attack_id`.
   - The session report (`wait_for_report` or `get_report`) holds the verdict, judge reasoning, and the agent's recorded response for each finding.
   Ask the user only for `session_id` and `attack_id` and pull the rest yourself.

2. **Manual paste (fallback).** Ask the user to open the finding in the cabinet (`/sessions/{session_id}` page → click the failed attack row) and paste:
   - `session_id`
   - `attack_id`
   - `category` / `subcategory`
   - Attacker prompt (verbatim, the full text)
   - Agent's response (verbatim, including any leak)
   - Judge reasoning (`reason` field on the finding)

# Tasks

1. Read the current system prompt and tool definitions of the agent in this repo.
2. Identify the specific weakness that allowed this attack to succeed. Quote the exact phrasing in the system prompt or the exact tool description that the attacker exploited.
3. Propose concrete, minimal edits — quote the exact lines that should change and write the replacement inline. No "consider tightening this section" — show the diff.
4. Add a unit / integration test (in whichever test runner the repo already uses — pytest / vitest / jest / etc.) that:
   - Replays this exact attacker prompt against the agent.
   - Asserts the safe behaviour (e.g., a refusal phrase, or that no PII appears in the response, or that a specific tool wasn't called).
   - Names the test after the `attack_id` so it's traceable back to Vector.
5. **Do NOT over-correct.** Avoid blanket refusals that would harm normal traffic — the fix should be the narrowest change that closes this specific gap. If the proposed fix could break a normal use case, flag it and ask before applying.

# Verification before declaring done

- Run the new test and confirm it passes against the patched agent.
- Optional: suggest re-running the original Vector session against the patched agent to confirm the finding is now PASS — point the user at the cabinet "Re-run" button (or the relevant MCP tool) rather than crafting a brand-new session.
