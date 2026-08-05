---
name: doc-prd
description: |
  Write a feature PRD (Product Requirements Document): problem, audience, scope, functional
  and non-functional requirements, goals with quantified targets, decisions and trade-offs,
  dependencies, risks, acceptance criteria and test strategy. Works from existing sources
  (meeting transcript, code, prior docs) or through a structured interview. Use when: (1) a
  feature needs its product-level requirements written down, (2) a discussion must become
  requirements a team can build against, (3) the "why and what" of a feature has to be
  separated from the "how". Keywords: "prd", "product requirements", "requisitos",
  "documento de requisitos", "feature prd", "requisitos funcionais".
---

# doc-prd

Generate a **feature** PRD: the document that answers *"por que e o quê?"*, at product
level, without descending into implementation.

> **Not to be confused with `prd-writer`** from the `sdd-skills` package. That one writes a
> **product** PRD in English, with 9 sections, feature IDs, a dependency graph and execution
> waves. This one writes a **feature** PRD in Portuguese, with functional and non-functional
> requirements, decisions with trade-offs, risks and test strategy. Different altitude,
> different shape. They coexist.

Output language: **Portuguese by default**, or the `language` parameter.

## Base docs (read when relevant)

- Operating modes and the extraction contract: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/modos-fonte-e-entrevista.md`
- Traceability and the sidecar format: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/rastreabilidade.md`
- Document altitudes: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/alturas-de-documentos.md`

The output skeleton is in `references/template-prd.md`. The interview guide, with the
question bank organised by stage, is in `references/guia-de-entrevista.md`.

## INPUT

| Parameter | Required | Default | Meaning |
| --- | --- | --- | --- |
| `output_path` | no | `docs/PRD.md` | Where the PRD goes |
| `sources` | no | — | Artifacts to extract from. Triggers FONTE mode |
| `rfc_path` / `fdd_path` / `adrs_folder` | no | — | Upstream documents to consolidate from and link |
| `min_functional_requirements` | no | — | Minimum number of functional requirements |
| `language` | no | `pt-BR` | Output language |
| `export_json` | no | `ask` | `true`, `false` or `ask`. Emits the structured JSON alongside the document |
| `acceptance_criteria` | no | — | Path or inline checklist validated in Phase 4 |
| `trace_sidecar` | no | `true` | Emit `docs/.trace/PRD.jsonl` |

## OUTPUT

- The PRD at `output_path`
- The structured JSON, when requested — same content, English keys, Portuguese values.
  Schema in `references/schema-prd.json`
- The traceability sidecar, when enabled

---

## Ordering note

When the package also contains an RFC, FDD and ADRs, **write the PRD last**. With the
decisions closed and the technical design settled, the PRD becomes a consolidation rather
than a guess, and it stops contradicting the documents below it.

Writing it first is fine when nothing else exists yet — that is the ENTREVISTA case.

---

## EXECUTION

### Phase 1 — Determine the mode and gather material

Follow the mode contract in the base doc.

In FONTE mode, read every source in full and classify each item. For the PRD:

| Class | Where it goes |
| --- | --- |
| `DECIDIDO` | Requisitos funcionais e não funcionais; decisões e trade-offs |
| `DESCARTADO` | **Fora de escopo**, named explicitly |
| `ADIADO` | **Fora de escopo**, named, with the phase it returns in |
| `EM ABERTO` | Riscos or dependências, depending on what it blocks |
| `SECUNDÁRIO` | Usually omitted; it belongs in the FDD |

The *Fora de escopo* section is where the quality of the reading shows. A discarded idea
that reappears as a requirement means the sources were skimmed, not read.

In ENTREVISTA mode, follow `references/guia-de-entrevista.md`: 12 stages, one question at a
time, a 3-to-6-line summary and a confirmation at the end of each stage.

### Phase 2 — Consolidate from upstream documents

When `rfc_path`, `fdd_path` or `adrs_folder` are given, read them and pull up:

- from the ADRs → *Decisões e trade-offs* (one line of decision, one of justification, one
  of trade-off, plus the link — never the ADR's full analysis)
- from the RFC → scope boundaries, risks
- from the FDD → non-functional requirements that have real numbers, and the shape of the
  test strategy

Pull **up**, never across: the PRD states the requirement, the FDD states how it is met.

### Phase 3 — Write

Follow `references/template-prd.md`. Section-specific requirements:

**Objetivos e métricas** — every objective gets a metric and a numeric target. An objective
without a number is an intention. When the sources give no number, ask; if the user defers,
apply a default and mark it `(hipótese)`.

**Escopo** — both halves. *Fora de escopo* names at least the discarded and deferred items
found in the sources, each with the reason.

**Requisitos funcionais** — each one gets an ID (`RF-01`), a name, a one-sentence
description, a step-by-step main flow, alternative flows and exceptions, predicted errors,
and a priority. Respect `min_functional_requirements`. Describe **what** the system does, not
how it is built: "o sistema entrega a notificação com garantia de ao menos uma entrega" is a
requirement; "o worker faz polling na tabela outbox a cada 2 segundos" is FDD.

**Requisitos não funcionais** — grouped by category (performance, disponibilidade, segurança
e autorização, observabilidade, confiabilidade, compatibilidade, compliance). Numbers or
clear norms; never adjectives alone.

**Decisões e trade-offs** — each decision with its justification and the cost it carries.
Link the ADR when one exists.

**Dependências** — technical, organizational and external, each one specific: who needs to
deliver what, and why it blocks.

**Riscos** — each with probability, impact, mitigation (a list when there is more than one
action) and a contingency plan.

**Critérios de aceitação** — an objective checklist. "Funciona bem" is not a criterion;
"toda alteração de status gera um evento persistido com o id do pedido, o status anterior e o
timestamp" is.

**Testes e validação** — which test types are mandatory and what the validation approach is.

### Phase 4 — Validation (internal)

Run this checklist. Fix and re-run, up to 3 iterations. Report what survives.

- [ ] All mandatory sections present and non-empty
- [ ] Every objective has a metric **and** a numeric target
- [ ] Every functional requirement has ID, description, main flow and priority
- [ ] `min_functional_requirements` respected
- [ ] Non-functional requirements cover at least performance and availability
- [ ] *Fora de escopo* contradicts nothing in scope, and names the discarded and deferred items
- [ ] Every decision carries a justification **and** a trade-off
- [ ] Every risk has probability, impact, mitigation and contingency
- [ ] Acceptance criteria are objective and verifiable
- [ ] Nothing contradicts the linked RFC, FDD or ADRs
- [ ] Hypotheses are visibly marked
- [ ] Altitude guard passes (see below)
- [ ] `acceptance_criteria`, when provided, satisfied item by item

**Altitude guard.** The PRD has leaked into FDD territory if it names a table, a column, a
class, an endpoint path, a header, an error code or a polling interval. Restate the item as
a capability and let the FDD carry the mechanism.

### Phase 5 — Write and verify

1. Write `output_path`.
2. When `export_json` is `ask`, ask the user after presenting the document. When `true`,
   write it alongside. Keys in English, values in the output language, no empty fields.
3. Write the sidecar when enabled: `PRD-FR-NN`, `PRD-NFR-NN`, `PRD-OBJ-NN`, `PRD-OUT-NN`,
   `PRD-RISCO-NN`.
4. **Read back** the file and confirm the mandatory sections are present.
5. Report: path, requirement counts, objectives with targets, out-of-scope items, and
   anything left as a hypothesis.

---

## ALWAYS

- A metric and a number on every objective.
- Name the discarded and deferred items in *Fora de escopo*.
- A trade-off on every decision.
- Probability, impact, mitigation and contingency on every risk.
- Mark hypotheses visibly.
- One question at a time in interview mode.

## NEVER

- Record a discarded or deferred item as a requirement.
- Ship an objective without a numeric target.
- Put table, column, class, endpoint or error-code names in the PRD.
- Use a double question in the interview.
- Write an acceptance criterion that cannot be checked.
- Restate an ADR's full analysis; link it.

## EDGE CASES

- **The sources describe the solution but never the problem:** ask. A PRD whose problem
  section was reverse-engineered from the solution justifies whatever was already built, and
  is worth very little.
- **No number anywhere for the objectives:** offer 2 or 3 plausible targets with their
  implications, and mark the chosen one as a hypothesis if the user is unsure.
- **A requirement that is really two:** split it. A requirement with an "and" in the middle
  gets tested as one thing and shipped as half.
- **The user wants the PRD before the technical documents:** fine. Run in ENTREVISTA mode and
  note in the document which items are hypotheses pending technical confirmation.
- **An existing PRD is present at `output_path`:** read it, preserve the IDs already
  assigned, and report what changed rather than silently overwriting.
