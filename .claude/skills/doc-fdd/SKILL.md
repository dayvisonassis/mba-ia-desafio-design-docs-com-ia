---
name: doc-fdd
description: |
  Write an FDD (Feature Design Document): the implementation-level specification of a
  feature, detailed enough for a developer to start coding. Covers detailed flows, public
  contracts with example payloads, an error matrix, resilience strategy, observability, and
  how the feature integrates with the existing codebase. Use when: (1) a technical approach
  is settled and needs to become an actionable spec, (2) a developer needs contracts, error
  codes and flows before implementing, (3) a feature must be documented against an existing
  system. Keywords: "fdd", "feature design", "design doc", "especificação técnica",
  "contratos", "como implementar", "technical spec".
---

# doc-fdd

Generate an FDD: the most technical document of the package. The test it must pass is
concrete — **a developer reads it and starts coding, without asking what a field means, what
an endpoint returns, or what happens when the third retry fails.**

Output language: **Portuguese by default**, or the `language` parameter.

## Base docs (read when relevant)

- Operating modes and the extraction contract: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/modos-fonte-e-entrevista.md`
- Traceability and the sidecar format: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/rastreabilidade.md`
- Document altitudes: `https://github.com/dayvisonassis/docs-skills/blob/main/docs/alturas-de-documentos.md`

The output skeleton is in `references/template-fdd.md`. Read it before writing.

## INPUT

| Parameter | Required | Default | Meaning |
| --- | --- | --- | --- |
| `output_path` | no | `docs/FDD.md` | Where the FDD goes |
| `sources` | no | — | Artifacts to extract from. Triggers FONTE mode |
| `codebase_root` | no | repo root | Code to analyse for the integration section |
| `rfc_path` / `adrs_folder` / `prd_path` | no | — | Upstream documents to build on and link |
| `error_code_prefix` | no | — | Prefix for error codes, e.g. `WEBHOOK_` |
| `integration_section` | no | `auto` | `true` forces the *Integração com o sistema existente* section; `auto` includes it whenever a codebase is present |
| `min_contracts` | no | — | Minimum number of public contracts to specify |
| `language` | no | `pt-BR` | Output language |
| `acceptance_criteria` | no | — | Path or inline checklist validated in Phase 5 |
| `trace_sidecar` | no | `true` | Emit `docs/.trace/FDD.jsonl` |

## OUTPUT

- The FDD at `output_path`
- The traceability sidecar, when enabled

---

## What an FDD is for

The FDD answers **"como construir, em detalhe?"**. It assumes the reader already accepted
that the feature should exist (PRD) and already agreed with the approach (RFC/ADRs).

| Belongs in the FDD | Does not |
| --- | --- |
| Table and column names, indexes, endpoints, payloads, headers, status codes | Why the feature matters to the business (PRD) |
| Error codes and what triggers each one | Re-arguing alternatives already closed (ADR) |
| Timeout, retry and backoff values | Adoption metrics and success targets (PRD) |
| How this plugs into the code that already exists | |

When a decision is questioned during writing, do not reopen it in the FDD. Link the ADR and
implement it.

---

## EXECUTION

### Phase 1 — Determine the mode and gather material

Follow the mode contract in the base doc. In FONTE mode, read every source in full and
classify. For the FDD, `SECUNDÁRIO` items matter more than they do elsewhere: technical
details mentioned in passing (a header name, a timeout, a payload field) are exactly the
material of this document. `DESCARTADO` and `ADIADO` items go to *Escopo e exclusões* and
appear nowhere else.

Read the upstream documents when provided. The FDD builds on them and links them; it does
not restate them.

### Phase 2 — Map the existing codebase

Skip only when there is no codebase. This phase is what makes the FDD implementable rather
than generic.

1. Map the structure: modules, layers, where a new module would live.
2. Identify the **integration points** the feature actually touches. For each one, record
   the real path and what it does today:
   - the service or transaction the feature must hook into
   - the error classes and error-code convention already in use
   - authentication and authorization middleware
   - the centralized error handler
   - the logger and its structured-log conventions
   - the data schema and how migrations are done
   - the state machine or domain rules that constrain the feature
3. **Verify every path exists.** A path cited in an FDD that is not on disk is worse than no
   citation: it is a fabricated fact that a developer will act on.
4. Extract the naming and layering conventions the new code must follow. Reusing existing
   patterns is nearly always the right call, and it is a design decision worth stating.

### Phase 3 — Fill the gaps

Compare what the sources gave you against what the skeleton needs: flow steps, contracts,
error conditions, resilience numbers, observability signals, acceptance criteria.

Ask about the gaps, one question at a time. Where the user defers, apply the default and mark
it `(hipótese)` in the document. An invented timeout that is not marked as a hypothesis
becomes a production value.

### Phase 4 — Write

Follow `references/template-fdd.md`. Section-specific requirements:

**Fluxos detalhados** — step by step, end to end, with the decision points named. Cover the
main flow and the variations that change behavior (failure, retry exhaustion, concurrency,
duplicate delivery). Say where validation happens, where persistence happens, where the
transaction opens and closes. A Mermaid sequence or flowchart earns its place; `doc-mermaid`
can generate them from this document afterwards.

**Contratos públicos** — for each contract: route and method, headers with their semantics,
a **request example and a response example** as real JSON, and the status codes with what
each one means. Fields get their type and meaning, not just a name. Respect `min_contracts`.

**Matriz de erros** — a table of every predicted error: code, condition that triggers it,
HTTP status, message, and treatment. When `error_code_prefix` is set, every code carries it.
Codes are `SCREAMING_SNAKE_CASE` and describe the condition, not the layer.

**Estratégias de resiliência** — concrete numbers: timeouts, retry count, backoff formula
with its base and ceiling, what happens after the last attempt, circuit-breaker thresholds if
any, and the invariants that must hold under failure.

**Observabilidade** — all three, always: **métricas** (name, type, labels, and what a healthy
value looks like), **logs** (format, mandatory fields, level per event, and what must never
be logged — secrets, payloads with personal data), **tracing** (spans, propagation across
process boundaries, sampling).

**Integração com o sistema existente** — include when `integration_section` is `true`, or
`auto` and a codebase exists. For each integration point: the **real file path**, what it
does today, and precisely how the feature plugs into it — the method extended, the class
reused, the middleware applied, the convention followed. This is the section that separates
an FDD written for this system from an FDD that could have been written for any system.

**Critérios de aceite técnicos** — objectively verifiable, testable statements. Not "o
sistema deve ser confiável" but "após 5 tentativas falhas, o evento é movido para a DLQ com o
motivo da última falha persistido".

### Phase 5 — Validation (internal)

Run this checklist. Fix and re-run, up to 3 iterations. Report what survives.

- [ ] All mandatory sections present and non-empty
- [ ] Every contract has request example, response example and status codes
- [ ] `min_contracts` respected
- [ ] Every error code carries `error_code_prefix` when set
- [ ] Every error in the matrix is reachable from some flow, and every failure named in a
      flow appears in the matrix
- [ ] Resilience section has real numbers, not adjectives
- [ ] Observability covers metrics **and** logs **and** tracing
- [ ] Integration section names at least one real path per integration point, and **every
      path cited anywhere in the document exists on disk**
- [ ] No `DESCARTADO` or `ADIADO` item specified as if it were in scope
- [ ] Nothing contradicts the linked ADRs or RFC
- [ ] Hypotheses are visibly marked
- [ ] Altitude guard passes (see below)
- [ ] `acceptance_criteria`, when provided, satisfied item by item

**Altitude guard.** Cut and link instead of keeping:

- business justification for the feature → PRD
- re-litigation of a closed decision, with alternatives → ADR
- adoption or revenue metrics → PRD

Keeping a one-line pointer to the motivation is fine. A page of it is a leak.

### Phase 6 — Write and verify

1. Write `output_path`.
2. Write the sidecar when enabled: `FDD-FLUXO-NN`, `FDD-CONTRATO-NN`, `FDD-ERRO-NN`,
   `FDD-INT-NN`.
3. **Read back** the file and confirm the mandatory sections are present and the contracts
   section is complete.
4. Report: path, contracts specified, error codes defined, integration points named, and any
   item left as a hypothesis.

---

## ALWAYS

- Verify a file path on disk before citing it.
- Give every contract a request example **and** a response example.
- Put a real number on every timeout, retry and threshold.
- Cover metrics, logs and tracing.
- Mark hypotheses visibly.
- Reuse the existing codebase's conventions, and say which ones you are reusing.

## NEVER

- Cite a file that does not exist.
- Specify something that was discarded or deferred.
- Contradict a linked ADR.
- Replace a number with an adjective ("rápido", "confiável", "escalável").
- Log secrets, tokens or signatures. When the feature handles them, say explicitly that they
  must not be logged.
- Reopen a closed decision inside the FDD.

## EDGE CASES

- **No codebase available:** omit the integration section, or keep it describing the
  interfaces the feature expects. Never invent paths.
- **The sources leave a contract half-specified:** specify what they support, mark the rest
  as a hypothesis, and list the missing pieces at the end of the contract.
- **A source contradicts the code** (the meeting says the field is called X, the code says Y):
  the code wins for what exists today; the source wins for what will be built. Note the
  divergence explicitly.
- **The feature touches a transaction or a state machine:** be exact about boundaries. Say
  what is inside the transaction, what is outside, and what happens if the process dies
  between the two. This is where most real bugs are designed in.
- **Contracts exceed what one document can hold cleanly:** keep the full spec of the main
  contracts inline and move the remainder to a companion file, linked. Do not truncate
  silently.
