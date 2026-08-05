# Esqueleto de PRD de feature (modelo de saída)

Textos entre colchetes são instruções e não aparecem na saída.

```markdown
# PRD: [produto] [feature]

| | |
| --- | --- |
| **Versão** | [versão] |
| **Data** | [YYYY-MM-DD] |
| **Responsável** | [nome] |
| **Documentos relacionados** | [RFC](RFC.md), [FDD](FDD.md), [ADRs](adrs/) |

---

## Resumo

[O contexto da feature em um ou dois parágrafos: o que é, para quem, e qual valor entrega.]

---

## Contexto e problema

**Público-alvo**
- [quem usa ou é afetado]

**Cenários de uso**
- [situação concreta em que a feature entra em ação]

**Onde a feature será implantada**
- [sistema existente ou novo sistema, com o nome do sistema]

**Problemas priorizados**
- [problema, com impacto e prioridade. Números quando houver: custo, tempo, frequência,
  clientes afetados]

---

## Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| [objetivo com verbo de ação] | [o que se mede] | [número alvo] |

[Todo objetivo tem métrica e meta numérica. Meta marcada como `(hipótese)` quando não veio da
fonte.]

---

## Escopo

**Incluso**
- [item]

**Fora de escopo**
- [item descartado, com a razão do descarte]
- [item adiado, com a fase em que volta]

---

## Requisitos funcionais

### RF-01 [nome do requisito]

[O que o sistema tem que fazer, em uma frase.]

**Fluxo principal**
1. [passo]
2. [passo]

**Fluxos alternativos e exceções**
- [variação ou exceção]

**Erros previstos**
- [condição que leva a erro e o que o sistema faz]

**Prioridade:** [alta | média | baixa]

### RF-02 [nome do requisito]

[Repetir a estrutura.]

---

## Requisitos não funcionais

**Performance**
- [meta numérica]

**Disponibilidade**
- [meta numérica]

**Segurança e autorização**
- [controle de acesso, autenticação, auditoria]

**Observabilidade**
- [logs, métricas, tracing]

**Confiabilidade e integridade de dados**
- [garantias transacionais, entrega, consistência]

**Compatibilidade e portabilidade**
- [versionamento de API, empacotamento]

**Compliance**
- [obrigação regulatória ou contratual]

---

## Decisões e trade-offs

### Decisão: [decisão]

- **Justificativa:** [por que]
- **Trade-off:** [o que custa]
- **Registro:** [ADR-NNN](adrs/ADR-NNN-....md)

### Decisão: [decisão]

[Repetir.]

---

## Dependências

### [técnica | organizacional | externa]: [título]

[Quem precisa entregar o quê, e por que isso bloqueia.]

---

## Riscos e mitigação

### [risco em uma frase]

- **Probabilidade:** [baixa | média | alta]
- **Impacto:** [consequência esperada]
- **Mitigação:**
  - [ação]
  - [ação]
- **Plano de contingência:** [plano B]

---

## Critérios de aceitação

Checklist objetivo que define se a feature está pronta.

- [critério verificável]
- [critério verificável]

---

## Testes e validação

**Tipos de teste obrigatórios**
- [tipo, e o que ele cobre]

**Estratégia de validação**
- [abordagem: TDD nas regras críticas, QA manual por roteiro, validação exploratória]
```

## Notas sobre o preenchimento

**Objetivos.** Objetivo → métrica → meta. Os três, sempre. "Reduzir a carga de polling dos
clientes" é uma intenção; "reduzir em 90% as chamadas a `GET /orders` dos clientes
integrados" é um objetivo.

**Fora de escopo.** É a seção que prova que a fonte foi lida com cuidado. Cada item
descartado ou adiado na discussão entra aqui nomeado, com a razão. Se um deles reaparecer
como requisito, o documento está errado.

**Requisitos funcionais.** Descrevem o **que**, nunca o **como**. Se aparecer nome de tabela,
de endpoint ou intervalo de polling, o conteúdo é de FDD.

**Requisitos não funcionais.** Números ou normas claras. "Alta disponibilidade" não é
requisito; "99,9% de uptime mensal" é.

**Riscos.** Os quatro campos são obrigatórios. Mitigação com mais de uma ação vira lista de
subitens.

**Critérios de aceitação.** Cada critério precisa ser verificável por alguém que não
participou da conversa.
