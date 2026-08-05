# Esqueleto de FDD (modelo de saída)

Textos entre colchetes são instruções e não aparecem na saída.

```markdown
# FDD: [nome da feature]

| | |
| --- | --- |
| **Versão** | [versão] |
| **Data** | [YYYY-MM-DD] |
| **Responsável técnico** | [nome] |
| **Documentos relacionados** | [PRD](PRD.md), [RFC](RFC.md), [ADRs](adrs/) |

---

## 1. Contexto e motivação técnica

[Qual problema técnico a feature resolve, como ela se encaixa no sistema existente, quais são
os atores e onde ficam os limites do escopo.

Uma referência curta ao PRD basta para a motivação de negócio. Não a reescreva aqui.

Suposições e restrições explícitas, quando houver.]

---

## 2. Objetivos técnicos

- [objetivo com a medida ou o invariante que o comprova]
- [objetivo com a medida ou o invariante que o comprova]

---

## 3. Escopo e exclusões

**Incluído**
- [item]

**Excluído**
- [item descartado ou adiado, com a razão e, quando for o caso, a fase em que volta]

---

## 4. Fluxos detalhados

### 4.1 [Fluxo principal: nome]

1. [passo, dizendo onde valida, onde persiste, onde abre e fecha transação]
2. [passo]

### 4.2 [Fluxo alternativo ou de exceção: nome]

1. [passo]

### Diagramas

[Diagramas Mermaid quando ajudarem. Podem ser gerados a partir deste documento com a skill
doc-mermaid.]

---

## 5. Contratos públicos

### 5.1 `[MÉTODO] /rota`

[O que faz, em uma linha. Quem pode chamar.]

**Headers**

| Header | Obrigatório | Semântica |
| --- | --- | --- |
| `[nome]` | [sim/não] | [significado] |

**Request**

```json
{}
```

| Campo | Tipo | Obrigatório | Semântica |
| --- | --- | --- | --- |
| `[campo]` | [tipo] | [sim/não] | [significado, limites, formato] |

**Response 2xx**

```json
{}
```

**Status codes**

| Código | Significado |
| --- | --- |
| `200` | [quando] |
| `400` | [quando] |

**Limites:** [rate limit, tamanho de payload, timeout esperado]

### 5.2 `[MÉTODO] /rota`

[Repetir a estrutura para cada contrato.]

---

## 6. Matriz de erros

| Código | Condição que dispara | HTTP | Mensagem | Tratamento |
| --- | --- | --- | --- | --- |
| `[PREFIXO_]NOME_DO_ERRO` | [o que aconteceu] | [status] | [mensagem retornada] | [o que o sistema faz: rejeita, reenfileira, move para DLQ, loga e segue] |

[Todo erro citado em algum fluxo aparece aqui, e todo erro daqui é alcançável por algum
fluxo.]

---

## 7. Estratégias de resiliência

**Timeouts**
- [operação: valor]

**Retry e backoff**
- [número de tentativas, fórmula do backoff com base e teto, jitter se houver]
- [o que acontece depois da última tentativa]

**Fallback e degradação**
- [comportamento quando a dependência está fora]

**Invariantes**
- [o que precisa continuar verdadeiro mesmo sob falha. Ex: "nenhum evento é perdido se o
  processo morrer entre a escrita e a publicação"]

---

## 8. Observabilidade

**Métricas**

| Métrica | Tipo | Labels | Valor saudável |
| --- | --- | --- | --- |
| `[nome]` | [counter \| gauge \| histogram] | [labels] | [faixa esperada] |

**Logs**
- Formato: [estruturado JSON, biblioteca usada]
- Campos obrigatórios: [correlation id, entidade, resultado]
- Nível por evento: [quais eventos em info, warn, error]
- **Nunca logar:** [segredos, assinaturas, tokens, dados pessoais]

**Tracing**
- Spans: [quais operações viram span]
- Propagação: [como o contexto atravessa fronteira de processo]
- Amostragem: [taxa]

**Dashboards e alertas**
- [painel ou alerta mínimo, com o gatilho]

---

## 9. Integração com o sistema existente

[Uma subseção por ponto de integração. Sempre com caminho real de arquivo, verificado.]

### 9.1 `caminho/real/do/arquivo.ext`

**O que faz hoje:** [comportamento atual]

**Como a feature se integra:** [o método que será estendido, a classe reutilizada, o
middleware aplicado, a convenção seguida. Concreto o bastante para o desenvolvedor abrir o
arquivo e saber o que fazer.]

### 9.2 `caminho/real/do/outro/arquivo.ext`

[Idem.]

---

## 10. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
| --- | --- | --- |
| [componente] | [versão] | [notas] |

**Garantias de compatibilidade**
- [o que continua funcionando igual, o que muda, como a transição acontece]

---

## 11. Critérios de aceite técnicos

- [afirmação verificável, testável, com número quando cabível]
- [afirmação verificável]

---

## 12. Riscos e mitigação

### [Risco]

- **Probabilidade:** [baixa | média | alta]
- **Impacto:** [consequência]
- **Mitigação:**
  - [ação]
- **Plano de contingência:** [plano B]
```

## Notas sobre o preenchimento

**Contratos.** Os exemplos de request e response são JSON de verdade, com valores
plausíveis, não `{}`. É a parte do documento que mais economiza tempo de quem implementa.

**Códigos de erro.** `SCREAMING_SNAKE_CASE`, descrevendo a condição e não a camada.
`WEBHOOK_ENDPOINT_NOT_FOUND` é bom; `WEBHOOK_SERVICE_ERROR` não diz nada.

**Números.** Timeout, retry, backoff, limites e valores saudáveis de métrica são números. Se
a fonte não deu o número, marque `(hipótese)` e siga, mas não troque o número por um
adjetivo.

**Integração com o sistema existente.** É a seção que prova que o documento foi escrito para
este sistema. Confirme cada caminho no disco antes de citá-lo. Um caminho inexistente aqui é
uma alucinação que alguém vai tentar abrir.

**Transações.** Quando a feature toca uma transação existente, seja explícito sobre a
fronteira: o que entra na transação, o que fica fora, e o que acontece se o processo morrer
no meio. É onde os bugs de verdade são desenhados.
