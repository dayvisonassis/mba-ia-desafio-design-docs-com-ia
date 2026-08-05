# Esqueleto de RFC (modelo de saída)

Documento conciso, de 2 a 4 páginas. Textos entre colchetes são instruções e não aparecem na
saída.

```markdown
# RFC: [título da proposta]

| | |
| --- | --- |
| **Autor** | [nome] |
| **Status** | [Rascunho \| Em revisão \| Aceito \| Rejeitado \| Substituído por RFC-NNN] |
| **Data** | [YYYY-MM-DD] |
| **Revisores** | [nome (papel), nome (papel), ...] |

---

## TL;DR

[3 a 6 linhas. Quem ler só isso precisa sair sabendo: qual o problema, qual a abordagem
proposta e qual o custo dela. Escreva esta seção por último.]

---

## Contexto e problema

[O que existe hoje e por que não basta. Números quando houver: volume, custo, latência,
frequência de incidente, pedido formal de cliente.

Quando o problema mora no código, cite o caminho real do arquivo e o que ele faz hoje.

Sem solução nesta seção.]

---

## Proposta técnica

[A forma da solução em nível de arquitetura: quais componentes existem, como o dado flui
entre eles, quais garantias o conjunto oferece.

Um diagrama se paga aqui.

Cada decisão fechada aparece em um parágrafo curto com link para o ADR correspondente:

> Optamos por [decisão] ([ADR-NNN](adrs/ADR-NNN-....md)) porque [razão em uma frase].

Pare antes de nome de coluna, payload, header e status code. Isso é FDD.]

### Escopo desta proposta

**Incluído**
- [item]

**Não incluído nesta fase**
- [item adiado, com a razão]

---

## Alternativas consideradas

### [Alternativa 1]

[O que era e por que era plausível.]

**Trade-off que levou ao descarte:** [motivo concreto.]

### [Alternativa 2]

[Idem.]

**Trade-off que levou ao descarte:** [idem.]

---

## Questões em aberto

### [Questão 1]

[O que ainda não está decidido.]

- **Bloqueia:** [o que não pode avançar sem isso, ou "nada, mas afeta X"]
- **Como resolver:** [medir em produção, spike de N dias, confirmar com fulano, aguardar dado]

### [Questão 2]

[Idem.]

- **Bloqueia:** [...]
- **Como resolver:** [...]

---

## Impacto e riscos

### Impacto

- **[Área afetada]:** [o que muda para quem. Consumidores atuais, operação, carga do time,
  migração de dado, custo de infraestrutura]

### Riscos

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| [risco] | [baixa \| média \| alta] | [consequência] | [ação] |

---

## Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001](adrs/ADR-001-....md) | [título] |
| [ADR-002](adrs/ADR-002-....md) | [título] |

---

## Referências

- [PRD, FDD, fontes da discussão]
```

## Notas sobre o preenchimento

**Tamanho.** De 2 a 4 páginas, algo entre 1000 e 2000 palavras. Quando estourar, o culpado
quase sempre é a proposta técnica tendo absorvido detalhe de implementação. Corte lá e linke
o FDD.

**TL;DR.** É a seção mais lida e a mais negligenciada. Escreva por último, quando o resto já
existe, e teste: entregue só ela a alguém e veja se a pessoa consegue explicar a proposta.

**Alternativas.** Duas, no mínimo, e reais. O valor está no trade-off, não na lista. "Fila
dedicada foi descartada" não diz nada; "fila dedicada foi descartada porque acrescentaria um
componente de infraestrutura que o time não opera hoje, para um volume que o MySQL atual
absorve" diz tudo.

**Questões em aberto.** É o que transforma o documento em um convite à revisão. Cada questão
precisa de um caminho de resolução, senão é só uma incerteza registrada.

**Revisores.** Em modo FONTE, são as pessoas que participaram da discussão, com o papel de
cada uma. São elas que têm contexto para revisar.
