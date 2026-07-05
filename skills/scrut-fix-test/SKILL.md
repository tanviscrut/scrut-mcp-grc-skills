---
name: scrut-fix-test
description: >-
  Remediate a failing Scrut continuous cloud test (CAT) from the user's IDE:
  find the failing test, gather remediation guidance and the flagged resources,
  help edit the infrastructure-as-code, trigger a re-scan, and confirm the fix.
  Use when a cloud/config compliance test is failing and the user wants to fix it
  where their code lives. Triggers include "fix this failing test", "remediate
  our public S3 buckets", "what's failing in our cloud tests", and "our
  encryption test is failing, help me fix it".
version: 1.0.0
---

# Scrut: fix a failing cloud test

Close the loop on a failing continuous automated test (CAT): understand it, fix
the underlying infrastructure code, re-scan, and verify — without leaving the
editor.

## What the terms mean

- **CAT test** — a Scrut continuous automated test that checks a cloud/config
  requirement (e.g. "S3 buckets must block public access").
- **`needs_attention`** — the status of a failing test (vs. `passing` or
  `ignored`).
- **Flagged resource** — a specific cloud resource that caused the test to fail;
  its type/config appears in the test detail payload.
- **Module type** — the test's module category. `scrut_run_test` needs it, but
  derives it automatically from the test when you omit it.
- **Re-scan (`scrut_run_test`)** — triggers a fresh evaluation of the test. It is
  **asynchronous**: it only starts the scan; results arrive later.

## Tools this skill uses

- `scrut_list_tests` — find failing tests (`test_status: "needs_attention"`).
- `scrut_get_test` — full detail for one test (`response_format: "json"` to see
  mapped controls, module type, and any flagged resources/config).
- `scrut_answer_question` — remediation guidance from the knowledge base.
- `scrut_get_document_content` — a related policy/evidence, if useful.
- `scrut_run_test` — trigger a re-scan after the fix (async).

## Procedure

1. **Find the failing test.**
   Call `scrut_list_tests` with `test_status: "needs_attention"`. If the user
   named a test, match it; otherwise present the failing list and confirm which
   one to work on.

2. **Get the detail.**
   Call `scrut_get_test` with the `test_id` and `response_format: "json"`. Note
   the mapped controls, the `moduleType`, and any **flagged resources** and their
   config in the payload — this is what you need to write a correct fix.

3. **Gather remediation guidance.**
   Call `scrut_answer_question` (e.g. *"How do we remediate <test name>?"*) for
   the fix steps. Optionally call `scrut_get_document_content` for a related
   policy or evidence if it clarifies the required end state.

4. **Fix it in the repo.**
   Locate the relevant infrastructure-as-code (Terraform, CloudFormation, etc.)
   for the flagged resource and propose the change. **Show the diff and confirm
   with the user before applying** — this edits real infrastructure code.

5. **Re-scan.**
   After the change is applied, call `scrut_run_test` with the `test_id` (pass
   `module_type` only if you want to override the auto-derived value). This
   **triggers** the scan and returns immediately (`triggered: true`); it does not
   return a pass/fail.

6. **Confirm — and account for lag.**
   After a short wait, poll `scrut_get_test` for the updated status.
   - Status `passing` → report the fix confirmed.
   - Status still `needs_attention` → **do not conclude the fix failed.** The
     re-scan runs in the background and the result may not have loaded/updated
     yet. Tell the user the scan may still be processing and they can **check
     again later or re-run**. Only call it a confirmed failure once a completed
     scan still shows `needs_attention` (e.g. the flagged resources are unchanged
     after the scan has clearly finished).

## Worked example

User: *"Our S3 public-access test is failing — fix it."*

1. `scrut_list_tests(test_status: "needs_attention")` → "S3 buckets must block
   public access" (`test_id: t_…`).
2. `scrut_get_test(test_id: t_…, response_format: "json")` → flagged buckets +
   `moduleType`.
3. `scrut_answer_question("How do we remediate public S3 buckets?")` →
   block-public-access guidance.
4. Edit Terraform to add `aws_s3_bucket_public_access_block` for the flagged
   buckets; show the diff; user approves.
5. `scrut_run_test(test_id: t_…)` → `triggered: true`.
6. Wait, then `scrut_get_test(test_id: t_…)`:
   - if `passing` → "Confirmed: the S3 public-access test now passes."
   - if still `needs_attention` → "Change applied and re-scan triggered. Results
     may still be processing — check again in a bit or re-run; I won't call this
     failed yet."

## Edge cases and honesty

- **Async is the main trap.** Never claim "fixed" until `scrut_get_test` confirms
  a completed passing scan. Distinguish "scan still processing / not updated yet"
  from "confirmed still failing".
- **Confirm before editing infra code.** Show the diff; do not silently rewrite
  IaC.
- **Report tool errors** (e.g. re-scan trigger failing) plainly instead of
  looping.
- **Availability:** if `scrut_run_test` is not present on this connection, do
  steps 1-4 and tell the user to re-run the scan from Scrut to verify.

## Changelog

- **1.0.0** — initial release: find failing test, detail + remediation, IaC fix
  with confirmation, async re-scan, and lag-aware verification.
