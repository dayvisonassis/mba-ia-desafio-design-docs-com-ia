---
name: doc-adr
description: |
  Write Architecture Decision Records in MADR format, one file per decision, either by
  extracting closed decisions from existing sources (meeting transcript, code, design
  thread) or by interviewing the user. Use when: (1) a technical decision has been made
  and needs to be recorded with its context, alternatives and consequences, (2) a meeting
  or thread closed several decisions that must become individual ADRs, (3) an existing
  decision is being replaced and needs a superseding record. Keywords: "adr", "decision
  record", "registro de decisão", "MADR", "decisão arquitetural", "architecture decision".
---

# doc-adr

Generate Architecture Decision Records in **MADR** format. One decision, one file, one
record of why the team chose what it chose and what it paid for that choice.

Output language: **Portuguese by default**, or the `language` parameter. These instructions
are in English; the generated documents are not.

## Base docs (read when relevant)

- Operating modes and the extraction contract: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/modos-fonte-e-entrevista.md`
- Traceability and the sidecar format: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/rastreabilidade.md`
- Document altitudes (what belongs in an ADR vs. RFC vs. FDD): `https://github.com/dayvisonassis/docs-skills/blob/main/docs/alturas-de-documentos.md`

The full output skeleton is in `references/template-adr.md`. Read it before writing the
first ADR of a run.

## INPUT

| Parameter | Required | Default | Meaning |
| --- | --- | --- | --- |
| `output_folder` | no | `docs/adrs` | Where the ADR files go |
| `sources` | no | — | Artifacts to extract from. Triggers FONTE mode |
| `decisions` | no | — | Explicit list of decisions to record. When absent, derive from sources |
| `language` | no | `pt-BR` | Output language |
| `min_adrs` / `max_adrs` | no | — | Bounds on how many ADRs to produce |
| `acceptance_criteria` | no | — | Path or inline checklist validated in Phase 4 |
| `trace_sidecar` | no | `true` | Emit the sidecar for the whole set at `docs/.trace/ADRs.jsonl` |
| `start_number` | no | auto | First ADR number. Auto-detected from existing files |

## OUTPUT

- One file per decision: `<output_folder>/ADR-NNN-titulo-em-kebab-case.md`
- An index at `<output_folder>/README.md` listing every ADR with its status
- The traceability sidecar, when `trace_sidecar` is on

---

## What is and is not an ADR

An ADR records **one decision**. This is the rule that carries the most weight, because
violating it is easy and it quietly destroys the value of the whole set.

| Belongs in an ADR | Does not |
| --- | --- |
| The choice made, stated as a decision | The full implementation spec (that is FDD) |
| Why the context forced the choice | The business justification for the feature (that is PRD) |
| Alternatives that were genuinely on the table | Alternatives invented to pad the section |
| What the team gained **and what it gave up** | Only the upside |

A record that decides the persistence pattern **and** the retry policy **and** the auth
scheme is not one ADR. It is three, and it must be split.

Signals of a real decision in a source: an explicit close (*"fechado então"*, *"vamos de"*),
a choice between named options, or a constraint accepted with a cost acknowledged.

---

## EXECUTION

### Phase 1 — Determine the mode and gather decisions

Follow the mode contract in the base doc. In short:

- `sources` given, or user pointed at artifacts → **FONTE**
- nothing to read → **ENTREVISTA**
- the usual case → **MISTO**

**In FONTE mode:** read every source in full, then extract candidate decisions and classify
each one as `DECIDIDO`, `DESCARTADO`, `ADIADO`, `EM ABERTO` or `SECUNDÁRIO`.

Only `DECIDIDO` becomes an ADR. This is not a formality:

- `DESCARTADO` items are **not** ADRs. They are the *Alternativas Consideradas* section of
  the ADR that beat them.
- `ADIADO` and `EM ABERTO` items are **not** ADRs. They belong in the RFC's open questions.
  Writing an ADR for an undecided matter fabricates consensus that never happened.

**In ENTREVISTA mode:** for each decision, ask in this order, one question at a time — what
was decided, what problem forced it, what else was considered, what it costs.

### Phase 2 — Plan the set

Before writing any file, present a numbered plan and get confirmation:

```
ADR-001  Padrão Outbox no MySQL          [DECIDIDO] fonte: [09:22] Diego
ADR-002  Retry com backoff e DLQ         [DECIDIDO] fonte: [09:34] Bruno
ADR-003  HMAC-SHA256 com secret por endpoint  [DECIDIDO] fonte: [09:41] Sofia
```

Rules for the plan:

- **Numbering** continues from the highest existing `ADR-NNN` in `output_folder`. Numbers
  are never reused and never renumbered, even when an ADR is superseded.
- **Ordering** follows the order the decisions were closed in the source when that is
  knowable; otherwise, foundational decisions first (the ones other decisions depend on).
- **Titles** are the decision itself, not the topic. `outbox-no-mysql` is a decision;
  `persistencia-de-eventos` is a topic. Kebab-case, no accents in the filename, accents
  preserved in the H1 inside the file.
- If two candidates share a rationale and always move together, they are one ADR. If one
  candidate contains an "and", check whether it is two.

### Phase 3 — Write each ADR

Follow `references/template-adr.md`. Mandatory sections: **Status, Contexto, Decisão,
Alternativas Consideradas, Consequências**.

Per-section requirements:

- **Status** — one of `Proposto`, `Aceito`, `Rejeitado`, `Substituído por ADR-NNN`,
  `Obsoleto`. Include the date. A decision extracted from a closed meeting is `Aceito`.
- **Contexto** — the forces that made a decision necessary: constraints, existing system,
  volume, deadline, prior art in the codebase. Written so it still makes sense to someone
  reading it in two years with no memory of the meeting. No solution here.
- **Decisão** — a single declarative sentence in the active voice, then the specifics that
  define its boundary. *"Vamos usar o padrão Outbox, gravando o evento na mesma transação
  que altera o status do pedido."*
- **Alternativas Consideradas** — at least one real alternative, each with the trade-off
  that killed it. An alternative with no stated reason for rejection is decoration.
- **Consequências** — positive **and** negative, both mandatory. The negative block is what
  makes an ADR worth reading; an ADR whose consequences are all good is a sales pitch.
  Where a consequence carries a mitigation or a follow-up, say so.

When the decision touches existing code, name the real paths and say how. Verify each path
exists before writing it.

Cross-reference related ADRs with relative links (`[ADR-002](ADR-002-....md)`), and the RFC
when there is one.

### Phase 4 — Validation (internal)

Run this checklist. Fix and re-run, up to 3 iterations. If issues remain after 3, stop and
report them to the user.

**Per ADR**
- [ ] All five mandatory sections present and non-empty
- [ ] Exactly one decision recorded
- [ ] At least one real alternative, each with an explicit trade-off
- [ ] Consequences include at least one negative
- [ ] Status is a valid value and dated
- [ ] Filename matches `ADR-NNN-titulo-em-kebab-case.md`; `NNN` is zero-padded to 3
- [ ] Every file path cited exists on disk
- [ ] No implementation-level detail that belongs in the FDD (see altitude guard below)

**Across the set**
- [ ] Numbering is sequential with no gaps and no reuse
- [ ] No two ADRs decide the same thing, and none contradict each other
- [ ] Every `DECIDIDO` item from Phase 1 became an ADR, or its omission was agreed with the user
- [ ] No `DESCARTADO`, `ADIADO` or `EM ABERTO` item was recorded as a decision
- [ ] `min_adrs` / `max_adrs` respected
- [ ] Cross-references resolve to files that exist
- [ ] The index in `README.md` lists every ADR with a matching status
- [ ] `acceptance_criteria`, when provided, is satisfied item by item

**Altitude guard** — an ADR that contains a DDL statement, a full endpoint contract, or a
list of column names has absorbed FDD content. Cut it down to the decision and its
rationale; the detail belongs in the FDD.

### Phase 5 — Write files and verify

1. Write each ADR file.
2. Write or update `<output_folder>/README.md` with the index.
3. Write the sidecar when `trace_sidecar` is on, one line per ADR, `tipo: "Decisão"`.
4. **Read back** every file written and confirm it contains the five sections. If a file is
   empty or truncated, regenerate and write again.
5. Report to the user: files created, decisions covered, and any `DECIDIDO` item
   deliberately left out.

---

## ALWAYS

- One decision per ADR.
- At least one negative consequence in every ADR.
- Every alternative carries the trade-off that rejected it.
- Numbers are stable: never renumber, never reuse.
- Verify a file path exists before citing it.
- Supersede rather than rewrite: when a decision changes, the old ADR gets
  `Substituído por ADR-NNN` and stays; the new one gets a `Substitui ADR-NNN`.

## NEVER

- Record an undecided, deferred or discarded matter as a decision.
- Invent an alternative that nobody considered, to fill the section.
- Write only positive consequences.
- Renumber existing ADRs.
- Put implementation specification inside an ADR.
- Delete a superseded ADR. The history is the point.

## EDGE CASES

- **A source closes more than `max_adrs` decisions:** rank by architectural weight (how much
  of the system the decision constrains), record the top ones, and list the remainder for
  the user to decide. Do not silently drop them.
- **A decision was made but the reasoning was not recorded:** write the ADR with the
  Contexto marked `(hipótese)` and ask the user to confirm. Do not invent a rationale.
- **The source contradicts itself** (decided, then reversed later in the same meeting): the
  later statement wins. Note the reversal in Contexto — it is genuinely useful history.
- **A decision that only makes sense together with another** (e.g. at-least-once delivery
  and idempotency by event id): keep them as one ADR and say why they are inseparable.
- **`output_folder` already has ADRs from another author:** read them first. Match their
  formatting conventions where they do not conflict with MADR, and continue their numbering.
- **A decision reverses an existing ADR:** create the new ADR with `Substitui ADR-NNN`, and
  edit the old one's status to `Substituído por ADR-NNN`. Both files stay.
