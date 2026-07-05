---
name: scrut-answer-questionnaire
description: >-
  Draft answers to a security or vendor questionnaire from the organization's
  documented compliance posture in Scrut, with a source citation per answer and
  an explicit gap flag when coverage is missing. Use when the user pastes or
  points at a set of security questions (vendor security review, CAIQ, DDQ, RFP
  security section). Triggers include "answer this questionnaire", "fill this
  vendor security review", "respond to these security questions", and "draft an
  answer to this from our policies".
version: 1.0.0
---

# Scrut: answer a security questionnaire

Turn a list of security questions into drafted answers grounded in what the org
has actually documented, each with a citation, and honestly flag anything the
knowledge base cannot support. **Never fabricate an answer.**

## What the terms mean

- **Answer library** — the org's curated bank of previously **approved**
  question -> answer pairs that reviewers reuse. A match here is the highest-trust
  source and can be reused verbatim.
- **`scrut_answer_question`** — a retrieval-augmented answer synthesized over the
  org's policies, trust vault, and product docs. It returns an `answer`,
  `references` (the sources it drew from), and `isAnswered`.
- **`isAnswered`** — the knowledge base's confidence flag. `false` means it was
  **not** confident; treat it as "no supported answer", not as a weak answer.
- **Document content** — the exact text of a specific named policy or evidence,
  used to quote or cite a source precisely.

## Tools this skill uses

- `scrut_search_answer_library` — look for an approved prior answer to reuse.
- `scrut_answer_question` — synthesize an answer from the knowledge base.
- `scrut_get_document_content` — pull an exact document's text for a citation or
  verbatim quote.
- `scrut_search_documents` — optional: locate a document by name/keyword before
  reading it.

## Procedure

1. **Split the questionnaire into individual questions.** Preserve any numbering
   so answers map back cleanly.

2. **For each question, source an answer in this order:**
   a. **Reuse an approved answer first.** Call `scrut_search_answer_library` with
      the question (or its key terms). If an entry clearly matches, reuse its
      answer **verbatim** and cite it as an approved answer-library entry.
   b. **Otherwise synthesize.** Call `scrut_answer_question` with the question.
      Read `answer`, `references`, and `isAnswered`.
   c. **Cite precisely when needed.** If the answer leans on a specific document
      the user wants quoted or verified, call `scrut_get_document_content` (by
      name; use `scrut_search_documents` first if you only have a keyword) and
      cite that document.

3. **Decide coverage per question — this is the core of the skill.**
   - **Answered:** the answer library matched, or `scrut_answer_question`
     returned `isAnswered: true` with references. Include the draft answer **and**
     its citation(s).
   - **Needs human / no coverage:** `isAnswered` is `false`, or nothing supports
     the answer. **Do not write an answer.** Mark it for a human and, if useful,
     note what document or fact would be needed to answer it.

4. **Assemble the output.** For each question return: the question, the draft
   answer (or a clear "needs human — no supported source" marker), the cited
   source(s), and the coverage flag. End with a one-line coverage summary
   (e.g. "34 answered, 6 need a human").

## Worked example

User: *"Answer this vendor security questionnaire from our policies."* — includes
*"Do you encrypt customer data at rest?"*

1. Split into questions.
2. For the encryption question:
   - `scrut_search_answer_library("encrypt data at rest")` → no exact approved
     entry.
   - `scrut_answer_question("Do you encrypt customer data at rest?")` →
     `isAnswered: true`, references an Information Security / Encryption policy.
   - `scrut_get_document_content(name: "Encryption Policy")` → exact clause to
     cite.
3. Coverage: **answered** — include the drafted answer + "Source: Encryption
   Policy".
4. In the output that row shows the answer, the citation, and the answered flag;
   a question with `isAnswered: false` shows "needs human — no supported source".

## Edge cases and honesty (non-negotiable)

- **Never fabricate.** An unanswered question is a correct, useful result — a
  made-up answer is a trust failure. Honor `isAnswered: false`.
- **Prefer approved, verbatim answers** from the answer library over synthesized
  ones when they match.
- **Cite every answer.** No answer ships without a source reference.
- **Conflicting sources:** prefer the more recent source but flag the conflict
  rather than silently choosing.
- **Tool availability:** if the answer-library or content tools are not present
  on this connection, say so and fall back to `scrut_answer_question` alone,
  still applying the no-fabrication and citation rules.

## Changelog

- **1.0.0** — initial release: per-question answer-library reuse, RAG synthesis
  with confidence handling, precise document citations, and a strict
  no-fabrication / gap-flag policy.
