---
name: doc-tracker
description: |
  Build a traceability matrix mapping every item in a documentation package back to its
  origin in a source (meeting transcript, code, prior document), and audit the package for
  hallucination: items with no identifiable origin, cited file paths that do not exist, and
  invalid source locations. Use when: (1) a set of design docs needs a traceability table,
  (2) documentation produced with AI must be verified against its sources, (3) someone asks
  "where did this requirement come from?". Keywords: "tracker", "rastreabilidade",
  "traceability", "matriz de rastreabilidade", "auditoria de documentação", "verificar
  fontes", "alucinação".
---

# doc-tracker

Build the traceability matrix and audit the documentation package against its sources.

This skill has two jobs, and the second is the important one. Producing the table is
mechanical. **Finding the items that have no origin is the point** — those are the places
where the AI, or the author, filled a gap with something plausible instead of something true.

Output language: **Portuguese by default**, or the `language` parameter.

## Base docs (read when relevant)

- Traceability, the sidecar format and the ID conventions: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/rastreabilidade.md`
- Source location formats: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/modos-fonte-e-entrevista.md`

## INPUT

| Parameter | Required | Default | Meaning |
| --- | --- | --- | --- |
| `documents` | no | `docs/**/*.md` | Documents to trace |
| `sources` | no | — | The sources the documents claim to derive from |
| `output_path` | no | `docs/TRACKER.md` | Where the matrix goes |
| `codebase_root` | no | repo root | Root for verifying cited paths |
| `trace_dir` | no | `docs/.trace` | Sidecars to aggregate |
| `columns` | no | see below | Column set of the matrix |
| `min_coverage` | no | `0.8` | Coverage threshold that triggers a warning |
| `language` | no | `pt-BR` | Output language |
| `acceptance_criteria` | no | — | Path or inline checklist validated in Phase 5 |

Default columns: `ID`, `Documento`, `Tipo`, `Conteúdo (resumo)`, `Fonte`, `Localização`.

## OUTPUT

- The traceability matrix at `output_path`
- An audit report to the user: coverage, items with no origin, non-existent paths, invalid
  locations. The report goes in the conversation, not inside the matrix file, unless the user
  asks for it

---

## EXECUTION

### Phase 1 — Collect

1. Read every sidecar in `trace_dir`. Each line is an item already traced by the skill that
   produced it. This is the reliable path.
2. Read every document in `documents`.
3. Read every source in `sources`, in full. You cannot verify a location you have not read.

### Phase 2 — Extract items from the documents

For every document, identify the traceable items. A traceable item is a statement a reader
could challenge with *"says who?"*:

| Item type | Where it typically appears |
| --- | --- |
| Requisito Funcional | PRD |
| Requisito Não Funcional | PRD |
| Objetivo / Métrica | PRD |
| Exclusão (fora de escopo) | PRD, RFC, FDD |
| Decisão | ADR, PRD, RFC |
| Trade-off | ADR, RFC, PRD |
| Alternativa considerada | RFC, ADR |
| Questão em aberto | RFC |
| Restrição | any |
| Contrato | FDD |
| Erro previsto | FDD |
| Ponto de integração | FDD |
| Risco | PRD, RFC, FDD |

Reconcile against the sidecars: an item with a sidecar line inherits its ID and origin. An
item with no sidecar line goes to Phase 3 for origin resolution.

Prose that carries no claim — a section intro, a transition sentence — is not an item. Do not
inflate the matrix with them; it dilutes the coverage signal that makes this document useful.

### Phase 3 — Resolve the origin of untraced items

For each item without an origin, in this order:

1. **Search the sources.** The item may be there in different words. Search by concept, not
   by string.
2. **Search the code**, when the item is a technical fact about the existing system.
3. **Ask the user.** They may hold context the sources never recorded.
4. **Mark as `HIPOTESE`**, if the user confirms it is a deliberate assumption.
5. **Flag for removal.** If it survives none of the above, it has no origin. Report it.

Never write a vague or invented location to fill the cell. An empty cell is a finding; a
false location is a lie that buys unearned trust.

### Phase 4 — Audit

Three verifications, all reported to the user:

**4.1 Path existence.** Extract every file path cited across all documents and check it on
disk. Report each non-existent path with the document and line that cites it. A path in a
design doc is an instruction: someone will try to open it.

**4.2 Location validity.**
- Timestamps cited exist in the transcript, and the quoted speaker is the one who spoke there.
- Speaker names match the declared participants.
- Cited document sections exist in the referenced document.
- Cited code paths exist (covered by 4.1) and plausibly contain what is claimed.

**4.3 Coverage.** Percentage of identified items that have a matrix row, plus the
distribution by source. Below `min_coverage`, report it as a warning with the list of
uncovered items — the shortfall is a documentation problem, not a tracker problem.

### Phase 5 — Write and verify

1. Sort rows by document, then by ID. Stable order makes diffs readable across regenerations.
2. Write the matrix to `output_path` using `columns`.
3. **Read back** the file and confirm the table is complete and well-formed.
4. Validate `acceptance_criteria` when provided.
5. Report to the user, in this order:
   - coverage, with the numbers
   - **items with no identifiable origin** — the headline finding
   - non-existent paths
   - invalid locations
   - distribution by source

Findings come before statistics. A tracker with 100% coverage and four fabricated paths is a
worse document than one with 85% coverage and none.

---

## Matrix format

```markdown
# Tracker de Rastreabilidade

[Uma ou duas linhas explicando o que a tabela faz e como ler a coluna Localização.]

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | [resumo de uma linha] | TRANSCRICAO | `[09:17] Diego` |
| FDD-INT-01 | `docs/FDD.md` | Ponto de integração | [resumo de uma linha] | CODIGO | `src/modules/orders/order.service.ts` |
```

Conventions:

- **ID** stable and unique. Never renumbered.
- **Conteúdo** is one line. If it needs two, the item is probably two items.
- **Fonte** is a label: `TRANSCRICAO`, `CODIGO`, `DOC`, `ENTREVISTA`, `HIPOTESE`.
- **Localização** follows the format for its source type. For transcripts with timestamps,
  `[hh:mm] Nome`.

---

## ALWAYS

- Read the sources in full before verifying a single location.
- Check every cited path on disk.
- Report items with no origin as the headline finding.
- Keep IDs stable across regenerations.
- Keep one line per item.

## NEVER

- Invent a location to fill a cell.
- Pad the matrix with non-items to raise coverage.
- Report coverage without reporting the findings.
- Silently drop an item that has no origin — flagging it is the whole job.
- Renumber existing IDs.

## EDGE CASES

- **No sidecars exist:** normal for hand-written or externally produced packages. Extract
  from the documents and resolve origins in Phase 3. Say in the report that traceability was
  reconstructed rather than recorded, since reconstruction is less reliable.
- **An item has two origins** (stated in the meeting *and* visible in the code): pick the one
  that actually originated it — usually the source — and mention the other in the summary.
- **A source has no timestamps:** use the format for that source type from the base doc, and
  say in the header of the matrix which convention was used.
- **A document cites a path that exists but does not contain what is claimed:** report it. It
  is a subtler failure than a missing file and it misleads just as effectively.
- **Documents contradict each other:** not strictly a tracker concern, but you are the only
  skill reading all of them at once. Report the contradiction.
- **The package is very large:** keep the full matrix; it is a reference document, not a
  narrative. Split by document into sections only if the file becomes unreadable.
