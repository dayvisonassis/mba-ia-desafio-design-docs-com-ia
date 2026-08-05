---
name: doc-mermaid
description: |
  Generate Mermaid diagrams from a technical document (FDD, HLD, RFC, spec or architecture
  notes), selecting only the diagrams that genuinely increase comprehension, and emitting
  them in a single self-contained Markdown file with syntax that renders without errors.
  Use when: (1) a technical document needs visual support for flows, algorithms or
  contracts, (2) someone asks for diagrams of a design doc, (3) an architecture must be
  visualised from written material. Keywords: "mermaid", "diagrama", "diagrams",
  "fluxograma", "sequence diagram", "visualizar arquitetura", "diagramas do FDD".
---

# doc-mermaid

Generate Mermaid diagrams from a technical document.

Two rules carry this skill. **First: only diagrams that significantly increase
comprehension** — a diagram that restates a bulleted list adds noise and costs the reader
attention. **Second: zero fabrication** — a diagram that shows a component the document never
mentions is a confident lie in a format people trust more than prose.

## INPUT

| Parameter | Required | Default | Meaning |
| --- | --- | --- | --- |
| `source_document` | **yes** | — | The technical document to read |
| `output_folder` | no | `docs/mermaid` | Where the file goes |
| `max_diagrams` | no | `10` | Hard ceiling. Typical range is 6 to 8 |
| `embed` | no | `false` | When true, insert the diagrams into `source_document` instead of a separate file |

## OUTPUT

- **One** Markdown file: `<output_folder>/<nome-do-documento>-diagrams.md`, with every
  diagram embedded as a ```mermaid block. Never separate `.mmd` files, never one file per
  diagram.
- When `embed` is true, the diagrams go into the source document at the relevant sections.

Syntax guardrails and safe templates per diagram type are in `references/sintaxe-mermaid.md`.
**Read that file before writing any diagram code** — it is the difference between diagrams
that render and diagrams that throw parse errors.

---

## Language

The output matches the **language of the source document**. Detect it and follow it.

- Proper orthography, with accents and cedilla, everywhere: titles, descriptions, notes, and
  **inside node labels** (Mermaid handles UTF-8).
- Technical terms stay in English: Service, Worker, Gateway, Store, Queue, Redis, Kafka, API,
  REST, Container, Database, Span, Logger.
- No emojis.

---

## EXECUTION

### Phase 1 — Read the document in full

Extract, without deciding anything yet:

- external actors and systems
- input and output channels: endpoints, events, queues
- internal processes with clear steps
- conditional decisions: modes, flags, strategies
- public contracts: interfaces, data structures, message formats
- error handling and fallback mechanisms
- **items marked as excluded or out of scope** — these must never appear in a diagram

Detect the language while reading.

### Phase 2 — Filter by significance

For each diagram candidate, ask:

1. Does it explain the end-to-end main flow?
2. Does it clarify something difficult or non-obvious — a conditional algorithm, a state
   machine, a retry or fallback mechanism, distributed coordination?
3. Does it illustrate an important architectural decision — operating modes, strategy
   selection, storage backends, degradation?
4. Does it show a public contract that integrators need?
5. Does it visualise relationships between components or entities?

**Eligible** when the answer is yes to at least one **and** the diagram meaningfully reduces
ambiguity. Otherwise, skip it. Skipping is a valid, frequent and correct outcome.

### Phase 3 — Choose the type

| The document describes | Use |
| --- | --- |
| endpoints or events with external actors, ordered in time | `sequenceDiagram` |
| an algorithm, internal steps, a state machine | `flowchart TD` |
| modes, flags, fallback, alternative strategies side by side | `flowchart LR` |
| interfaces, structs, published types | `classDiagram` |
| stable relationships between entities or messages | `erDiagram` |

Several diagrams of the same type are fine when each serves a different purpose — three
sequence diagrams for the happy path, the failure path and the retry path is a good set.

### Phase 4 — Prune

- Ceiling: `max_diagrams`. Typical: 6 to 8. Floor: 1.
- Never two diagrams saying the same thing.
- A flow with more than 8 steps gets grouped into 5 or fewer logical nodes.
- A diagram with more than about 10 nodes gets split into two complementary views.

Priority when there are more candidates than slots: main flow, then the most complex logic,
then the key architectural variation, then the critical public contract, then the resilience
pattern.

### Phase 5 — Write

Structure of the output file, with headings translated into the document's language:

```markdown
# Diagramas Mermaid - [nome da feature]

## Visão Geral
[2 a 4 frases sobre o objetivo do sistema, com base no documento.]

## Elementos Identificados
### Fluxos externos
### Processos internos
### Variações de comportamento
### Contratos públicos

## Diagramas

### [Título do diagrama]

[Parágrafo de 3 a 5 frases: o que o diagrama representa, quando consultá-lo, por que ele
importa. O tipo do diagrama entra naturalmente na descrição.]

```mermaid
[código]
```

**Notas**
- [ponto de explicação]

---
```

The title names the diagram, not its type: *"Fluxo Principal"*, not *"Sequence Diagram -
Fluxo Principal"*.

The document ends after the last diagram. No *Análise*, *Racional*, *Decisões de Design* or
*Garantias de Consistência* sections — they duplicate the source document at a worse
altitude.

Apply `references/sintaxe-mermaid.md` while writing, not afterwards.

### Phase 6 — Internal review

Silent quality gate. Re-read the source document and the generated file, then find and fix:

- elements present in the diagrams but absent from the document — **fabrication, remove**
- significant elements from the document missing where a diagram claims to cover them
- wrong technologies, names or versions
- relationships that contradict the document
- missing accents, anywhere, including node labels
- technical terms wrongly translated
- syntax patterns from the guardrails file
- excluded or out-of-scope items appearing in a diagram
- redundancy between diagrams

Fix everything found. Do not document this review in the output file, and do not add an
"issues resolved" section.

### Phase 7 — Validate and report

- [ ] One file, in `output_folder`, correctly named
- [ ] Language matches the source document, accents correct everywhere
- [ ] Between 1 and `max_diagrams` diagrams
- [ ] Every diagram passes at least one significance test
- [ ] No redundancy, no fabrication, no excluded items
- [ ] Node labels of 3 words or fewer
- [ ] Every ```mermaid block is closed
- [ ] No guardrail violations
- [ ] Structure ends after the last diagram

**Read back** the written file to confirm it is complete. Then report: detected language,
file path, diagram count, why those diagrams were chosen, and which candidates were skipped
and why.

---

## ALWAYS

- Read the source document in full before generating anything.
- Match the source document's language, with correct accents inside node labels.
- Keep technical terms in English.
- One single output file.
- Read `references/sintaxe-mermaid.md` before writing diagram code.
- Skip a diagram rather than pad the set.

## NEVER

- Invent an element the document does not contain.
- Diagram something the document marks as out of scope.
- Create separate `.mmd` files or one file per diagram.
- Use emojis.
- Put accents, spaces or special characters in node **identifiers** (labels are fine).
- Reference the source document's section numbers inside labels or notes.
- Add analysis or rationale sections at the end.

## EDGE CASES

- **The document is too thin for meaningful diagrams:** generate the one or two that hold up,
  and state plainly what information would be needed for the rest. Do not fill the gap.
- **More than `max_diagrams` genuine candidates:** apply the Phase 4 priority order, and list
  the ones left out so the user can ask for them.
- **The document contains contradictions:** diagram the version the document states most
  explicitly, and report the contradiction. Do not silently pick one.
- **`source_document` does not exist:** ask for the correct path. Do not guess from the
  folder.
- **`embed` is true:** insert each diagram at the section it illustrates, keep the surrounding
  text intact, and read the file back to confirm nothing was displaced.
