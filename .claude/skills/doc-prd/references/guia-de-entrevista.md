# Guia de entrevista para PRD de feature

Usado no modo ENTREVISTA. As regras gerais de ritmo (uma pergunta por vez, resumo e
confirmação ao fim de cada etapa, defaults marcados como hipótese) estão em
[modos-fonte-e-entrevista.md](https://github.com/dayvisonassis/docs-skills/blob/main/docs/modos-fonte-e-entrevista.md).

## Mensagem inicial

> Vou te fazer algumas perguntas para entender a necessidade desta feature, o problema que
> ela resolve, o objetivo de negócio e onde ela vai rodar. No final eu gero o PRD no formato
> padrão e, se você quiser, também em JSON estruturado. Podemos começar com um resumo rápido
> da feature e por que ela é necessária agora?

## As 12 etapas

### 1. Contexto e visão geral

- Qual é o produto ou sistema em que essa feature entra?
- Essa feature pertence a um sistema que já existe ou faz parte de um sistema novo?
- Quem é o público-alvo?
- Em duas ou três frases, qual é o objetivo de negócio desta feature?

### 2. Problema e oportunidade

- O que acontece hoje que torna essa feature necessária?
- Tem um exemplo real recente, com números aproximados? Custo, tempo perdido, erro
  operacional, impacto no cliente.
- O que já foi tentado e não funcionou?

### 3. Objetivos e métricas de sucesso

- Que resultado mensurável você quer alcançar?
- Qual métrica representa esse resultado?
- Qual é a meta alvo dessa métrica?

> Ligue sempre objetivo → métrica → meta. Sem os três, a etapa não fechou.

### 4. Escopo

- O que precisa obrigatoriamente estar pronto nesta entrega?
- O que está explicitamente fora de escopo?

### 5. Requisitos funcionais

Para cada requisito, nesta ordem:

- Qual o nome do requisito?
- Em uma frase simples, o que o sistema tem que fazer?
- Qual o fluxo principal, passo a passo?
- Quais as variações comuns e as exceções?
- Em que condições o sistema bloqueia ou retorna erro?
- Qual a prioridade?

### 6. Requisitos não funcionais

- Performance esperada. Exemplo: p95 abaixo de 150 ms.
- Disponibilidade esperada. Exemplo: 99,9%.
- Segurança e controle de acesso. Autenticação, auditoria, permissão por papel.
- Observabilidade. Logs estruturados, métricas, tracing.
- Confiabilidade. Que operações precisam ser transacionais.
- Compliance, acessibilidade, compatibilidade.

### 7. Arquitetura e abordagem

Só o suficiente para o PRD registrar a abordagem. O detalhe é do HLD e do FDD.

- Essa feature roda onde? Monólito, microsserviço, worker, agente.
- A comunicação é síncrona, assíncrona ou ambas?
- Vai usar fila, mensageria ou streaming?
- Quais integrações externas são necessárias?

> Se o usuário não tiver uma visão de arquitetura, ofereça 2 ou 3 opções com prós e contras.

### 8. Decisões e trade-offs

- Que decisões técnicas já foram tomadas?
- Por que cada uma foi decidida assim?
- Qual o trade-off de cada uma?

### 9. Dependências

- Existe algo que precisa chegar de outro time ou de outra área? Design, política comercial,
  aprovação legal.
- Existe algo técnico que precisa estar pronto antes?

### 10. Riscos e mitigação

- Quais são os principais riscos?
- Para cada risco: probabilidade, impacto, mitigação e plano de contingência.

> Mitigação com mais de uma ação vira lista de subitens.

### 11. Critérios de aceitação

- Quais frases objetivas definem que a feature está pronta?

> Evite "funciona bem". Um bom critério: "toda alteração de preço gera auditoria persistida
> com quem alterou, o preço anterior e o timestamp".

### 12. Testes e validação

- Quais tipos de teste são obrigatórios? Unitário, integração, segurança, carga.
- Qual será a abordagem de validação? TDD, QA manual guiado por roteiro, validação
  exploratória.

## Checagens antes de gerar

- Cada objetivo tem métrica e meta alvo.
- Todo requisito funcional tem nome, descrição, fluxo principal e prioridade.
- Requisitos não funcionais incluem pelo menos performance e disponibilidade.
- Fora de escopo não contradiz o que está incluso.
- Toda decisão técnica relevante tem justificativa e trade-off.
- Cada dependência é específica: quem entrega o quê.
- Cada risco tem probabilidade, impacto, mitigação e plano de contingência.
- Os critérios de aceitação são verificáveis.
- Os tipos de teste obrigatórios estão definidos.
