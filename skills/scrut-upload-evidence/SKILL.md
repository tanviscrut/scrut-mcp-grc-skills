---
name: scrut-upload-evidence
description: >-
  File an artifact into Scrut as evidence. Use when the user has produced or
  received a document (access review, config export, vendor report, screenshot,
  signed policy) or a link and wants it attached to the right evidence item in
  Scrut. Triggers include "file this as evidence", "attach this to <evidence or
  control>", "upload this access review to Scrut", and "add this report as
  evidence".
version: 1.0.0
---

# Scrut: upload and attach evidence

Attach a file or a link to the correct evidence item in Scrut, so an artifact the
user just produced becomes filed evidence without leaving the conversation.

## What "evidence" means here

- **Evidence item** — a record in Scrut that collects proof for one or more
  controls. It has an `evidenceId`, an `evidenceName`, a status, an owner, and a
  review date. You attach documents to it; you do not create it here.
- **Document object** — the object returned by `scrut_upload_file` after a file
  is uploaded (it carries the stored file's `name`, `url`, etc.). It must be
  passed through **verbatim** to the attach step.
- **Evidence link** — an external URL attached in place of an uploaded file (for
  artifacts that already live somewhere accessible).
- **Note** — free text stored alongside the attachment (context, not the date).
- **Evidence date** — when the artifact is *as of*. It defaults to now; only set
  it to record a different date (e.g. an access review actually run last week).

## Tools this skill uses

- `scrut_list_evidence` — find and confirm the target evidence item.
- `scrut_upload_file` — upload a local file, returns a document object.
- `scrut_attach_evidence_document` — attach the document object (or a link) to
  the evidence item.

## Procedure

1. **Identify the target evidence item — do not guess.**
   Call `scrut_list_evidence`. Match against what the user said (an evidence name,
   a control code like "CC6.3", or a description like "our access reviews").
   - Exactly one clear match → use its `evidenceId`, and tell the user which item
     you matched.
   - Zero or more than one plausible match → show the candidate names + ids and
     **ask the user which one** before attaching. Never attach to the wrong item
     just to complete the task.
   - Large programs are paginated: if the list reports more results, page through
     with `offset`/`limit` before concluding there is no match.

2. **Decide file vs. link.**
   - **File** the user provided or you generated → read its bytes, base64-encode
     them, and call `scrut_upload_file` with `filename` (include the extension),
     `content_base64`, and `content_type` (e.g. `application/pdf`). Keep the
     returned `documentObject`.
   - **Link** to an artifact that already lives somewhere → skip the upload; you
     will pass the URL directly in the next step.

3. **Attach to the evidence item.**
   Call `scrut_attach_evidence_document` with:
   - `evidence_id` — from step 1.
   - **exactly one of**: `document_object` (the object from step 2) *or*
     `evidence_link` (the URL).
   - `note` — include if the user gave context worth storing.
   - `evidence_date` — omit unless the artifact is as-of a different day than
     today (it defaults to now).

4. **Confirm.**
   Report: the evidence item you attached to (name + id), what was attached
   (filename or link), the note, and the effective date. If step 1 was uncertain
   and the user chose, restate the choice.

## Worked example

User: *"Attach this Q3 access review PDF to our user access reviews evidence."*

1. `scrut_list_evidence` → one item named "User Access Reviews"
   (`evidenceId: ev_…`). Confirm the match with the user if any ambiguity.
2. `scrut_upload_file` with `filename: "q3-2026-access-review.pdf"`,
   `content_base64: <…>`, `content_type: "application/pdf"` → `documentObject`.
3. `scrut_attach_evidence_document` with `evidence_id: ev_…`,
   `document_object: <documentObject>`, `note: "Q3 2026 user access review"`.
4. "Attached **q3-2026-access-review.pdf** to **User Access Reviews**
   (`ev_…`) with the note 'Q3 2026 user access review', dated today."

## Edge cases and honesty

- **Ambiguous or missing target** → ask; list candidates. Attaching to the wrong
  evidence is worse than pausing to confirm.
- **Write may not be available on every connection.** If `scrut_upload_file` or
  `scrut_attach_evidence_document` returns an error, report it plainly and stop —
  do not silently retry or fabricate a success.
- **One source only.** `scrut_attach_evidence_document` takes a document object
  *or* a link, not both. Decide which in step 2.
- **You are not reading the file into Scrut's knowledge.** This skill files an
  artifact; it does not index its contents.

## Changelog

- **1.0.0** — initial release: find evidence, upload a file or attach a link,
  with note and optional as-of date.
