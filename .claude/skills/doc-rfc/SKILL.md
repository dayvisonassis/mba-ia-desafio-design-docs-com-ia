---
name: doc-rfc
description: |
  Write an RFC (Request for Comments): a concise technical proposal submitted to the team
  for review, covering the proposed approach, the alternatives that were rejected, and the
  questions still open. Use when: (1) a technical approach needs to be circulated for
  review before implementation, (2) a design discussion must be consolidated into a
  reviewable document, (3) a set of ADRs needs a narrative that ties them together.
  Keywords: "rfc", "request for comments", "proposta técnica", "design proposal",
  "technical proposal", "submeter para revisão".
---

# doc-rfc

Generate an RFC: the document a team reads to answer *"do we agree with this approach, and
what still needs deciding?"*.

Output language: **Portuguese by default**, or the `language` parameter.

## Base docs (read when relevant)

- Operating modes and the extraction contract: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/modos-fonte-e-entrevista.md`
- Traceability and the sidecar format: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/rastreabilidade.md`
- Document altitudes: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/alturas-de-documentos.md`

The output skeleton is in `references/template-rfc.md`.

## INPUT

| Parameter | Required | Default | Meaning |
| --- | --- | --- | --- |
| `output_path` | no | `docs/RFC.md` | Where the RFC goes |
| `sources` | no | — | Artifacts to extract from. Triggers FONTE mode |
| `adrs_folder` | no | `docs/adrs` | ADRs to link from *Decisões relacionadas* |
| `author` / `reviewers` / `status` | no | — | Metadata. Reviewers default to the participants found in the sources |
| `language` | no | `pt-BR` | Output language |
| `max_pages` | no | `4` | Length ceiling. See the concision guard |
| `acceptance_criteria` | no | — | Path or inline checklist validated in Phase 4 |
| `trace_sidecar` | no | `true` | Emit `docs/.trace/RFC.jsonl` |

## OUTPUT

- The RFC at `output_path`
- The traceability sidecar, when enabled

---

## What an RFC is for

An RFC **proposes and opens for review**. That verb pair defines everything about the
document.

| An RFC contains | It does not contain |
| --- | --- |
| The approach, at architecture level | The implementation spec (that is the FDD) |
| Alternatives that were really considered, and why they lost | A survey of everything possible |
| **Questions still open** | Pretend certainty about them |
| Impact and risk of adopting it | Business justification for the feature (that is the PRD) |
| Links to the ADRs that closed each decision | The full reasoning of each decision (that is the ADR) |

The section that most distinguishes a real RFC from a summary is **Questões em aberto**. A
proposal with nothing open is not being submitted for review; it is being announced. If the
sources show nothing open, say so explicitly and check with the user, because it is unusual.

### Relationship to the ADRs

Write the ADRs first when you can. The RFC then narrates and links them:

> Optamos pelo padrão Outbox no MySQL ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)) porque a
> escrita do evento entra na mesma transação que muda o status do pedido.

One paragraph plus a link. The RFC does not repeat the ADR's alternatives analysis; it
mentions the alternatives at the level a reviewer needs to judge the proposal.

---

## EXECUTION

### Phase 1 — Determine the mode and gather material

Follow the mode contract in the base doc. In FONTE mode, read every source in full and
classify each item. The mapping into this document is direct and it is the core of the job:

| Class | Where it goes in the RFC |
| --- | --- |
| `DECIDIDO` | Proposta técnica, with a link to the ADR |
| `DESCARTADO` | **Alternativas consideradas**, with the trade-off that rejected it |
| `ADIADO` | Escopo, as an explicit non-goal for this phase |
| `EM ABERTO` | **Questões em aberto** |
| `SECUNDÁRIO` | Usually omitted; it belongs in the FDD |

Read `adrs_folder` if it exists, and build the list of decisions to link.

In ENTREVISTA mode, ask in this order, one question at a time: what problem, what approach,
what else was on the table and why it lost, what is still undecided, what breaks if this
ships.

### Phase 2 — Assemble the metadata

- **Autor** — from `author`, or ask.
- **Revisores** — in FONTE mode, the participants identified in the sources, with their
  roles when stated. These are the people who were in the discussion; they are the natural
  reviewers.
- **Status** — `Rascunho`, `Em revisão`, `Aceito`, `Rejeitado`, `Substituído por RFC-NNN`.
  A document being circulated is `Em revisão`.
- **Data** — the date of writing, or the date of the discussion when that is what matters.

### Phase 3 — Write

Follow `references/template-rfc.md`. Mandatory sections: metadata, TL;DR, contexto e
problema, proposta técnica, alternativas consideradas, questões em aberto, impacto e riscos,
decisões relacionadas.

Section-specific requirements:

- **TL;DR** — 3 to 6 lines. Someone who reads only this must be able to state the problem,
  the approach and what it costs. Write it last, however high it sits in the document.
- **Contexto e problema** — what exists today and why it is not enough. Cite the real code
  when the problem lives there. No solution yet.
- **Proposta técnica** — the shape of the solution: components, how data flows between them,
  the guarantees it provides. A diagram earns its place here. Stop before column names,
  payload schemas and status codes.
- **Alternativas consideradas** — at least two that were genuinely on the table, each with
  the trade-off that killed it. Prefer alternatives the sources actually record. An
  alternative with no stated reason for rejection is decoration.
- **Questões em aberto** — at least the ones the sources leave open. For each: what is
  undecided, what it blocks, and how it will be resolved (measure, spike, wait for data,
  ask a specific person). An open question with no path to resolution is a wish.
- **Impacto e riscos** — what changes for whom: existing consumers, operations, the team's
  workload, data migration. Risks with a probability and a mitigation.
- **Decisões relacionadas** — links to the ADRs. Verify each linked file exists.

### Phase 4 — Validation (internal)

Run this checklist. Fix and re-run, up to 3 iterations. Report what survives.

- [ ] All mandatory sections present and non-empty
- [ ] Metadata complete: author, status, date, reviewers
- [ ] TL;DR standalone: states problem, approach and cost in 3 to 6 lines
- [ ] At least 2 alternatives, each with an explicit trade-off
- [ ] At least 2 open questions, each with a resolution path
- [ ] No `DESCARTADO` or `ADIADO` item presented as part of the proposal
- [ ] Every ADR link resolves to a file that exists
- [ ] Every file path cited exists on disk
- [ ] Within `max_pages` (see the concision guard)
- [ ] Altitude guard passes (see below)
- [ ] `acceptance_criteria`, when provided, satisfied item by item

**Concision guard.** Estimate pages at roughly 500 words per page. Over `max_pages`, do not
trim evenly: find the section that absorbed FDD detail, cut that, and link the FDD instead.
The oversized section is nearly always *Proposta técnica*.

**Altitude guard.** The RFC has leaked into FDD territory if it contains any of: a full
endpoint contract with request and response payloads, a table schema or column list, error
codes, specific timeout or retry values presented as spec, or a file-by-file implementation
plan. Mentioning that a management API will exist is RFC. Specifying its payloads is FDD.

The RFC has leaked into PRD territory if it argues why the feature matters to customers or
carries adoption metrics. Reference the PRD instead.

### Phase 5 — Write and verify

1. Write `output_path`.
2. Write the sidecar when enabled: one line per alternative (`RFC-ALT-NN`), per open question
   (`RFC-OPEN-NN`) and per risk (`RFC-RISCO-NN`).
3. **Read back** the file and confirm the mandatory sections are present.
4. Report: path, estimated length, number of alternatives, open questions and linked ADRs.

---

## ALWAYS

- Write the TL;DR last.
- Keep at least two open questions honest and visible.
- Give every alternative the trade-off that rejected it.
- Link ADRs instead of restating their reasoning.
- Verify a path or link before writing it.
- Name the reviewers who were actually in the discussion.

## NEVER

- Present a discarded or deferred item as part of the proposal.
- Fabricate an alternative to fill the section.
- Ship an RFC with an empty open-questions section without flagging it to the user.
- Copy the FDD's contracts, schemas or error codes into the RFC.
- Exceed `max_pages` by padding the technical proposal.

## EDGE CASES

- **No ADRs exist yet:** write the RFC anyway, with *Decisões relacionadas* listing the
  decisions and a note that the ADRs are pending. Suggest running `doc-adr` first, since the
  RFC gets sharper when the decisions are already closed.
- **The sources record only one alternative:** use it, and add a plausible second marked
  `(alternativa plausível, não discutida na fonte)`. Never present an invented alternative as
  debated.
- **Nothing was left open in the sources:** say so in the section, and ask the user whether
  something is open that the discussion did not capture. Do not invent open questions, and do
  not silently ship an empty section.
- **The proposal changes an existing public contract:** that belongs in *Impacto e riscos*,
  named explicitly, with the compatibility strategy.
- **A superseding RFC:** set `Substitui RFC-NNN` in the metadata and update the old one's
  status. Keep both.
