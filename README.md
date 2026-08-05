# Da Reunião ao Documento: como este pacote de design docs foi produzido

> Este README documenta o **processo**. O enunciado original do desafio está preservado em
> [`docs/DESAFIO.md`](docs/DESAFIO.md).

## Sobre o desafio

O material de entrada era uma transcrição literal de 55 minutos de reunião técnica, com cinco
participantes discutindo um Sistema de Webhooks de Notificação de Pedidos, e o código de um
Order Management System em produção que não tem nenhum mecanismo de notificação externa. A
saída precisava ser um pacote de design docs acionável o suficiente para o time começar a
codar: PRD, RFC, FDD, entre 5 e 8 ADRs, um tracker de rastreabilidade e este README. Sem tocar
em uma linha do código da aplicação.

A dificuldade real do desafio não é volume de escrita, é **discernimento**. A transcrição
mistura, no mesmo fluxo de conversa, decisões fechadas, ideias rejeitadas, pontos adiados para
fases futuras, questões deixadas em aberto e detalhes técnicos soltos. Identificar o que **não**
entra é tão importante quanto identificar o que entra, e é exatamente onde uma IA sem
instrução específica falha: ela lê "a gente podia mandar email quando falhar" e escreve um
requisito funcional de notificação por email, que a reunião descartou explicitamente três
segundos depois. Todo o processo abaixo foi montado em torno de impedir esse erro.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (Opus 5) | Ferramenta única de produção. Leu a transcrição e o código, extraiu e classificou os itens, escreveu os documentos e executou as validações automáticas |
| **[`docs-skills`](https://github.com/dayvisonassis/docs-skills)** | Pacote de 7 skills criado **neste desafio**, uma por tipo de documento. É onde moram os prompts customizados |
| **[`sdd-skills`](https://github.com/dayvisonassis/sdd-skills)** | Pacote próprio anterior, usado como referência de convenção: estrutura de `SKILL.md`, `references/` sob demanda, fase de validação interna |
| **Prompts do curso** | Modelos de entrevista de PRD, FDD e HLD, e os geradores de diagrama C4 e Mermaid, usados como base dos esqueletos de saída |

## Workflow adotado

A decisão que definiu tudo foi **não escrever os documentos direto no chat**. Em vez disso,
converti os prompts do curso em skills versionadas e produzi os documentos invocando-as.

Três motivos:

1. O README exige mostrar prompts customizados e iterações. Skills são exatamente isso, em
   forma versionada, revisável e reexecutável.
2. Os prompts do curso são **entrevistas** (uma pergunta por vez até preencher o esqueleto).
   Aqui não havia ninguém para entrevistar: a informação já existia na transcrição e no
   código. Era preciso um modo de operação diferente.
3. O trabalho sobrevive ao desafio. As skills ficam instaladas e servem aos próximos projetos.

### Fase A: construir as skills

Sete skills publicadas em [`docs-skills`](https://github.com/dayvisonassis/docs-skills), uma
por tipo de documento, cada uma com `SKILL.md` enxuto e `references/` carregadas sob demanda.
Três ideias que os prompts originais não tinham:

**Modo duplo.** Cada skill opera em modo FONTE (extrai de artefatos) ou ENTREVISTA (o Q&A do
curso), decidindo pelo que recebeu. O caso normal é misto: extrai o que a fonte sustenta,
pergunta o resto.

**Classificação antes de geração.** Cada item extraído é rotulado antes de qualquer escrita.
É a defesa contra o erro descrito acima.

**Rastreabilidade como auditoria.** Cada skill emite um sidecar `.jsonl` com a origem de cada
item, e o `doc-tracker` os agrega e valida mecanicamente.

### Fase B: produzir os documentos

Ordem deliberada, do mais concreto para o mais abstrato:

```
transcrição + código
   │
   ├─► ADRs      as decisões são o esqueleto de tudo
   │     │
   │     ├─► RFC     a proposta se apoia nelas e as linka
   │     │     │
   │     │     └─► FDD    o desenho técnico se constrói sobre a proposta
   │     │            │
   │     │            └─► PRD    por último: vira consolidação, não adivinhação
   │     │
   └─────┴─► TRACKER    varre tudo e audita
                 │
                 └─► README
```

O PRD por último foi o que mais rendeu. Com ADRs, RFC e FDD prontos, escrevê-lo é consolidar
o que já está decidido, e ele para de contradizer os documentos abaixo dele.

## Prompts customizados

Os três abaixo são os que mudaram o resultado. Estão reproduzidos como estão nas skills.

### 1. Classificação antes de geração

O prompt que impede uma ideia rejeitada de virar requisito. Vive em
[`docs/modos-fonte-e-entrevista.md`](https://github.com/dayvisonassis/docs-skills/blob/main/docs/modos-fonte-e-entrevista.md)
e é carregado por todas as skills geradoras.

```markdown
### 3. Classificação (o passo que mais protege o documento)

Nem tudo que foi dito vira requisito. Classifique **cada item extraído**:

| Classe | Significado | Para onde vai |
| --- | --- | --- |
| `DECIDIDO` | Fechado, com acordo explícito na fonte | Requisito, decisão, ADR |
| `DESCARTADO` | Levantado e rejeitado | Fora de escopo; alternativa descartada no RFC |
| `ADIADO` | Aceito em princípio, jogado para depois | Fora de escopo, marcado como fase futura |
| `EM ABERTO` | Discutido sem conclusão | Questões em aberto do RFC |
| `SECUNDÁRIO` | Detalhe técnico mencionado de passagem | FDD, se sustentar a implementação; senão, descartar |

Sinais linguísticos úteis em transcrições: *"fechado então"*, *"vamos de"*, *"decidido"* →
`DECIDIDO`. *"não vamos fazer"*, *"descarta"*, *"não compensa"* → `DESCARTADO`. *"fase 2"*,
*"depois a gente vê"*, *"fica pra próxima"* → `ADIADO`. *"precisa confirmar"*, *"não sei
ainda"*, *"vamos medir"* → `EM ABERTO`.

Na dúvida entre `DECIDIDO` e `EM ABERTO`, escolha `EM ABERTO` e pergunte. O custo de
registrar uma decisão que não foi tomada é muito maior que o de fazer uma pergunta.

**Um item `DESCARTADO` ou `ADIADO` nunca aparece como requisito.** Ele aparece nomeado na
seção de exclusões, e é isso que prova que a leitura da fonte foi cuidadosa.
```

Esse bloco sozinho produziu a seção "Fora de escopo" do PRD com 6 itens nomeados e as 5
questões em aberto do RFC.

### 2. A escada de resolução de origem

O prompt que decide o que fazer quando um item não tem fonte. Vive em
[`skills/doc-tracker/SKILL.md`](https://github.com/dayvisonassis/docs-skills/blob/main/skills/doc-tracker/SKILL.md).

```markdown
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
```

É por causa dessa escada que o pacote tem exatamente 5 itens marcados como `HIPOTESE`, em vez
de 5 itens com origem inventada.

### 3. A seção de integração com o sistema existente

O prompt que força o FDD a ser deste sistema, e não de qualquer sistema. Vive em
[`skills/doc-fdd/SKILL.md`](https://github.com/dayvisonassis/docs-skills/blob/main/skills/doc-fdd/SKILL.md).

```markdown
**Integração com o sistema existente** — include when `integration_section` is `true`, or
`auto` and a codebase exists. For each integration point: the **real file path**, what it
does today, and precisely how the feature plugs into it — the method extended, the class
reused, the middleware applied, the convention followed. This is the section that separates
an FDD written for this system from an FDD that could have been written for any system.
```

Combinado com a regra `Verify a file path on disk before citing it`, produziu as 10
subseções da seção 9 do FDD, com 13 caminhos reais verificados.

## Iterações e ajustes

Quatro momentos em que a saída veio errada ou insuficiente e precisou de correção.

### 1. Reaproveitar o `prd-writer` existente não funcionou

**O que veio:** a intenção inicial era instalar o `prd-writer` que já existe no meu pacote
`sdd-skills`, testado e em uso, em vez de escrever um novo.

**Por que estava errado:** a comparação seção a seção contra os requisitos do desafio mostrou
que faltavam **5 das 12 seções obrigatórias** (requisitos não funcionais, decisões e
trade-offs, riscos, estratégia de testes, escopo incluso). O "Dependency Graph" dele é
dependência entre features, não dependência técnica. E ele instrui explicitamente
`Write the entire PRD in English` e `NEVER: Include extra sections`. Era um PRD de **produto**;
o desafio pede um PRD de **feature**.

**Correção:** criei o `doc-prd` a partir do esqueleto do curso, que bate 12 de 12. Os dois
coexistem sem colisão de nome.

**Resultado:** evitou entregar um documento estruturalmente incompatível, e evitou quebrar um
fluxo que já uso.

### 2. Caminhos de arquivo que ainda não existem

**O que veio:** a primeira versão dos ADRs dizia "a lógica de processamento **vive** em
`src/modules/webhooks/webhook.worker.ts`", no presente.

**Por que estava errado:** a validação automática de caminhos acusou 3 arquivos inexistentes.
E o critério de aceite do desafio diz que nenhum arquivo mencionado pode ser inexistente. Mas
o FDD **precisa** dizer onde o código novo vai morar, então a regra literal e a natureza do
documento colidem.

**Correção:** criei a convenção `**(novo)**`, declarada em um bloco no topo do FDD, e reescrevi
as frases para "a ser criada em `src/worker.ts`". Verifiquei em modo multilinha que as três
citações carregam o marcador.

**Resultado:** 13 caminhos citados existem e podem ser abertos; 3 estão marcados como criação.
Nenhuma ambiguidade sobre o que é código atual e o que é proposta.

### 3. O RFC estourou o limite de páginas, e a culpa não era da prosa

**O que veio:** 2133 palavras, cerca de 5 páginas, contra as 2 a 4 exigidas.

**Por que estava errado:** a guarda de concisão da própria `doc-rfc` reprovou. A instrução da
skill manda não cortar por igual, e sim procurar a seção que absorveu detalhe de FDD.

**Correção:** medi por seção. `Proposta técnica` era a maior, mas a guarda de altura passou
limpa: nenhum payload, nenhum status code, nenhum header. Não havia vazamento, havia gordura.
O corte que rendeu de verdade foi a **tabela de "Decisões relacionadas", que reproduzia linha a
linha o índice de `docs/adrs/README.md`**. Virou lista compacta com link para o índice.

**Resultado:** 1743 palavras de prosa. E um defeito descoberto na minha própria skill: a
heurística de 500 palavras por página conta linha de tabela como prosa, e superestima
documentos densos em tabela. Está anotado para a próxima versão do pacote.

### 4. O reuso de padrões esconde duas dívidas que a reunião não viu

**O que veio:** a primeira leitura da decisão de "reuso máximo" (`[09:30] Larissa`) produziu
uma lista de consequências só positivas: curva de aprendizado zero, consistência, nada a
alterar.

**Por que estava errado:** um ADR cujas consequências são todas boas não foi analisado, foi
anunciado. Fui ao código conferir o que o reuso custa de fato.

**Correção:** duas descobertas que ninguém levantou na reunião, ambas em
[`src/shared/logger/index.ts`](src/shared/logger/index.ts). Primeiro, o logger tem
`base: { service: 'order-management-api' }` fixo, então os logs do worker sairiam identificados
como se fossem da API. Segundo, a lista `redactPaths` cobre `authorization`, `token` e
`password`, mas **não conhece `secret` nem `signature`** — num módulo cuja razão de existir é
manipular material secreto. Reusar o padrão dá falsa sensação de proteção.

**Resultado:** as duas viraram consequência negativa no
[ADR-006](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md), tarefa obrigatória na seção 9.7
do FDD e critério de aceite testável. É o tipo de achado que só aparece cruzando a fonte com o
código, e é o que separa documentação de transcrição reformatada.

## Validações automáticas executadas

O tracker não é só uma tabela; é o relatório de uma auditoria. Tudo abaixo foi verificado por
execução, não por leitura:

| Verificação | Resultado |
| --- | --- |
| Citações `[hh:mm] Nome` conferidas contra a transcrição | **112 de 112 válidas** |
| Caminhos de código conferidos no disco | **23 de 23 existem** |
| Identificadores duplicados no tracker | nenhum |
| Estrutura MADR completa nos 7 ADRs | 7 de 7 |
| Links entre documentos que resolvem | todos |
| Vazamento de altura (FDD dentro do RFC, FDD dentro do PRD) | nenhum |
| Arquivos de código alterados | **zero** |

## Como navegar a entrega

| Arquivo | O que contém |
| --- | --- |
| [`docs/PRD.md`](docs/PRD.md) | Por que e o quê: problema, público, escopo, 12 requisitos funcionais, métricas, riscos |
| [`docs/RFC.md`](docs/RFC.md) | Como propomos resolver, 4 alternativas descartadas e 5 questões em aberto |
| [`docs/adrs/`](docs/adrs/README.md) | 7 decisões, uma por arquivo, cada uma com alternativas e o que custou |
| [`docs/FDD.md`](docs/FDD.md) | Como construir: fluxos, 8 contratos, 12 códigos de erro, resiliência, observabilidade, integração com o código atual |
| [`docs/TRACKER.md`](docs/TRACKER.md) | De onde veio cada um dos 140 itens |
| [`docs/DESAFIO.md`](docs/DESAFIO.md) | Enunciado original, preservado |
| [`TRANSCRICAO.md`](TRANSCRICAO.md) | A fonte. Não alterada |

**Ordem sugerida de leitura**

1. [`docs/PRD.md`](docs/PRD.md) — entenda o problema e o escopo antes de qualquer solução.
2. [`docs/RFC.md`](docs/RFC.md) — a proposta em nível de arquitetura, e o que ficou em aberto.
3. [`docs/adrs/`](docs/adrs/README.md) — cada decisão em profundidade. Comece pelo
   [ADR-001](docs/adrs/ADR-001-outbox-no-mysql.md), que sustenta os demais.
4. [`docs/FDD.md`](docs/FDD.md) — o detalhamento. A seção 9 é a que amarra tudo ao código real.
5. [`docs/TRACKER.md`](docs/TRACKER.md) — consulte quando quiser saber a origem de qualquer
   afirmação dos documentos acima.

As skills que produziram este pacote estão em
[`.claude/skills/`](.claude/skills/) e publicadas em
[github.com/dayvisonassis/docs-skills](https://github.com/dayvisonassis/docs-skills).
