---
name: doc-process-readme
description: |
  Write the README that documents how a documentation package was produced with AI: tools
  used, workflow adopted, the custom prompts written, the iterations where the AI got
  something wrong and it had to be corrected, and how to navigate the delivery. Use when:
  (1) a docs package is finished and the process behind it must be recorded, (2) a README
  needs to explain a delivery rather than an application, (3) someone must reproduce the
  workflow that produced a set of documents. Keywords: "readme do processo", "process
  readme", "como foi produzido", "documentar o processo", "workflow de produção",
  "prompts customizados".
---

# doc-process-readme

Write the README that records **how** a documentation package was produced, not what the
software does.

The value of this document is honesty about the iterations. A README that reports a clean
first pass is either false or describes a shallow package: the first output of any AI on a
real source is generic, and the corrections are where the thinking happened.

Output language: **Portuguese by default**, or the `language` parameter.

## INPUT

| Parameter | Required | Default | Meaning |
| --- | --- | --- | --- |
| `output_path` | no | `README.md` | Where the README goes |
| `documents` | no | `docs/**/*.md` | The delivered package |
| `sources` | no | — | Sources the package derives from |
| `preserve` | no | — | Sections of the existing README to keep |
| `min_prompts` | no | `2` | Minimum custom prompts to show |
| `min_iterations` | no | `2` | Minimum concrete iterations to describe |
| `language` | no | `pt-BR` | Output language |
| `acceptance_criteria` | no | — | Path or inline checklist validated in Phase 4 |

## OUTPUT

- The README at `output_path`

---

## EXECUTION

### Phase 1 — Reconstruct the process

The process is not in the files; it is in what happened. Gather it from, in order of
reliability:

1. **The conversation itself** — the corrections, the rejected outputs, the questions asked
   and the decisions made. This is the primary source and it is usually the only place the
   iterations exist.
2. **`git log`** — commit order and messages show the real production sequence, which often
   differs from the planned one. That difference is worth reporting.
3. **The skills and prompts used** — the custom prompts are versioned artifacts, not
   recollections.
4. **The delivered documents** — what exists, and in what shape.
5. **The user** — ask about anything you cannot reconstruct. Never invent an iteration.

### Phase 2 — Select the material

**Prompts.** Pick the ones that changed the outcome, not the ones that were merely typed. A
prompt earns its place if removing it would have produced a visibly worse document. Show it
verbatim, in a code block. At least `min_prompts`.

**Iterations.** Pick the moments where the output was wrong or shallow and had to be
corrected. For each: what the AI produced, why it was wrong, what was done, and what changed
as a result. At least `min_iterations`.

The useful iterations are almost always of these kinds:

- the AI treated something discarded in the source as a requirement
- the output was generic — true of any system, specific to none
- two documents said the same thing at different altitudes
- a cited file path did not exist
- a number appeared with no origin

If none of these happened, say what did. What must not happen is a vague "refinamos o
documento algumas vezes", which reports nothing.

### Phase 3 — Write

Follow `references/template-readme.md`. Required sections:

- **Sobre o desafio / o projeto** — the task in your own words, 1 to 2 paragraphs
- **Ferramentas de IA utilizadas** — each with the role it played
- **Workflow adotado** — the order of production and why that order, including where the
  actual order departed from the plan
- **Prompts customizados** — at least `min_prompts`, verbatim, in code blocks
- **Iterações e ajustes** — at least `min_iterations`, concrete, with the correction and its
  effect
- **Como navegar a entrega** — file paths and a suggested reading order

When `preserve` is set, keep those sections from the existing README.

### Phase 4 — Validation (internal)

- [ ] All required sections present
- [ ] At least `min_prompts` prompts, in code blocks, verbatim
- [ ] At least `min_iterations` iterations, each naming the error, the fix and the effect
- [ ] Every tool listed actually played the role described
- [ ] Every path in the navigation section exists on disk
- [ ] Reading order is coherent with the package's altitudes
- [ ] Nothing claimed about the process that did not happen
- [ ] `acceptance_criteria`, when provided, satisfied item by item

### Phase 5 — Write and verify

1. When `output_path` exists, read it first. If it holds content the user may want (an
   original brief, a licence note), ask before replacing.
2. Write the file.
3. **Read back** and confirm the sections are present.
4. Report: path, prompts shown, iterations described, and anything you could not reconstruct.

---

## ALWAYS

- Reconstruct from evidence: the conversation, the git log, the artifacts.
- Show prompts verbatim.
- Describe iterations concretely: what was wrong, what was done, what changed.
- Verify every path in the navigation section.
- Ask before overwriting a README that holds content worth keeping.

## NEVER

- Invent an iteration that did not happen.
- Show a prompt that was not used.
- Report a clean first pass unless it genuinely happened.
- Claim a tool played a role it did not.
- Replace the concrete with the vague ("ajustamos e melhoramos o resultado").

## EDGE CASES

- **The process spanned several sessions:** reconstruct what you can from the artifacts and
  the git history, and ask the user about the rest. Say in the README that part of the
  process is reported from record rather than from memory, if that matters.
- **The existing README holds the original brief:** offer to keep it as a linked section or
  a separate file. Do not discard it without asking.
- **Fewer real iterations than `min_iterations`:** report the real ones and tell the user the
  minimum was not met. Do not manufacture the difference.
- **Several AI tools were used:** give each one the role it actually had. "Usei o Claude Code
  para ler o código e a transcrição, e o ChatGPT para revisar a redação do PRD" is useful;
  a bare list is not.
