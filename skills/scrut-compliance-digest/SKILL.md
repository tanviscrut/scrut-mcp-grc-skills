---
name: scrut-compliance-digest
description: >-
  Produce a leadership-ready snapshot of the organization's compliance program
  from Scrut. Use when the user wants a current-state summary for a board update,
  standup, exec review, or "are we on track" check across frameworks, controls,
  policies, evidence, and continuous tests. Triggers include "compliance
  digest", "where does our SOC 2 program stand", "give me a board update on
  compliance", and "how many policies are still in draft".
version: 1.0.0
---

# Scrut: compliance digest

Assemble a concise, paste-ready status summary of the compliance program:
readiness and next audits, program counts, and what is stale or unfinished.

## What the terms mean

- **Framework** — a compliance standard the org is pursuing (SOC 2, ISO 27001,
  GDPR, …), each with a `nextAuditDate`.
- **Control** — a requirement within a framework, with an owner and a
  per-framework status.
- **Policy status** — a policy's lifecycle state; **Draft** means not yet
  finalized/approved.
- **Evidence review date** (`reviewDate`) — when the evidence is next due for
  review; a date **in the past** means the evidence is **stale**.
- **Test status** — a continuous automated (CAT) test is `passing`,
  `needs_attention` (failing), or `ignored`.

## Tools this skill uses

All are read-only. Prefer `response_format: "json"` so you can count reliably.

- `scrut_list_frameworks` — frameworks + `nextAuditDate`.
- `scrut_list_controls` — controls (optionally per `framework_ids`).
- `scrut_list_policies` — policies + `policyStatus` + `reviewDate`.
- `scrut_list_evidence` — evidence + `evidenceStatus` + `reviewDate`.
- `scrut_list_tests` — tests + status; `test_status` filters (`passing`,
  `needs_attention`, `ignored`).

## Procedure

1. **Set scope.**
   If the user named a framework, resolve it with `scrut_list_frameworks` (match
   on name or `shortKey`) and note its `nextAuditDate`. Otherwise cover all
   enabled frameworks. Capture each framework's `frameworkId` for optional
   filtering.

2. **Pull the program data — page through everything.**
   Call each list tool with `response_format: "json"`. The JSON carries a
   pagination envelope (`total_count`, `has_more`, `next_offset`). **Loop with
   `offset = next_offset` until `has_more` is false** — do not summarize from
   only the first page, or the counts will be wrong.
   - `scrut_list_controls` (pass `framework_ids` when scoped to one framework)
   - `scrut_list_policies`
   - `scrut_list_evidence`
   - `scrut_list_tests` (get counts by status; you can call with
     `test_status: "needs_attention"` to count failing tests directly)

3. **Compute the numbers.**
   - **Next audit date** per in-scope framework.
   - **Counts:** controls, policies, evidence items, tests.
   - **Policies in Draft:** count where `policyStatus` is Draft.
   - **Stale evidence:** count where `reviewDate` is before today.
   - **Failing tests:** count with status `needs_attention`.

4. **Write the digest.**
   Lead with readiness (frameworks + next audit dates), then the counts, then a
   short **"What to watch"** (2-3 lines naming the biggest gaps — most Draft
   policies, most stale evidence, or the failing tests). Keep it tight and
   leadership-toned; it should paste straight into a doc, email, or Slack.

## Worked example

User: *"Give me a compliance digest for the board."*

1. `scrut_list_frameworks` (json) → SOC 2 (next audit 12 Aug 2026), ISO 27001
   (next audit 3 Nov 2026).
2. Page through `scrut_list_controls`, `scrut_list_policies`,
   `scrut_list_evidence`, `scrut_list_tests` (json) until each `has_more` is
   false.
3. Compute: 142 controls, 31 policies (4 Draft), 210 evidence items (6 past
   review), 58 tests (3 needs_attention).
4. Output:
   > **Compliance snapshot — 5 Jul 2026**
   > - SOC 2 — next audit **12 Aug**; ISO 27001 — next audit **3 Nov**.
   > - 142 controls · 31 policies · 210 evidence · 58 tests.
   > - **4 policies still in Draft**, **6 evidence items past review**, **3 tests
   >   need attention**.
   > **What to watch:** finalize the 4 Draft policies before the SOC 2 audit;
   > refresh the 6 stale evidence items; triage the 3 failing tests.

## Edge cases and honesty

- **Never invent numbers.** If a list call fails, say which part is unavailable
  rather than guessing a count.
- **Pagination is mandatory** for accurate totals — always follow `has_more`.
- **Empty results** may mean the connection is pointed at the wrong region/tenant
  or the user's role scopes the data — say so instead of reporting "0 program".
- **Summarize, don't dump.** The output is a digest; do not paste raw lists.

## Changelog

- **1.0.0** — initial release: framework/control/policy/evidence/test snapshot
  with next-audit dates, draft/stale/failing counts, and a "what to watch".
