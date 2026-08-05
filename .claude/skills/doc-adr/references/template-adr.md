# Esqueleto de ADR (formato MADR)

A saída deve seguir exatamente esta estrutura. Textos entre colchetes são instruções, não
devem aparecer no documento final.

```markdown
# ADR-[NNN]: [título da decisão, com acentuação normal]

## Status

[Proposto | Aceito | Rejeitado | Substituído por ADR-NNN | Obsoleto] em [YYYY-MM-DD]

[Quando aplicável, uma destas linhas:]
Substitui [ADR-NNN](ADR-NNN-....md)
Substituído por [ADR-NNN](ADR-NNN-....md)

## Contexto

[As forças que tornaram uma decisão necessária. Restrições técnicas, o que já existe no
sistema, volume esperado, prazo, obrigações de segurança ou de compliance, dívida técnica
relevante.

Escreva para alguém que vai ler isso daqui a dois anos sem ter participado da conversa. Cite
o que existe hoje no código quando for relevante, com o caminho real do arquivo.

Não apresente a solução aqui. O contexto é o problema.]

## Decisão

[Uma frase declarativa, na voz ativa, dizendo o que foi decidido. Depois, os detalhes que
delimitam a decisão: o que exatamente está dentro dela e onde ela para.]

[Quando a decisão tiver parâmetros fechados na fonte (um algoritmo, um número de tentativas,
um formato), liste-os. Quando os parâmetros ficaram para o detalhamento técnico, diga que
ficaram, e aponte o FDD.]

## Alternativas Consideradas

### [Alternativa 1]

[O que era. Por que era plausível.]

**Trade-off que levou ao descarte:** [o motivo concreto. Custo, complexidade operacional,
latência, acoplamento, prazo, risco de segurança.]

### [Alternativa 2]

[Idem.]

**Trade-off que levou ao descarte:** [idem.]

## Consequências

### Positivas

- [O que essa decisão passa a permitir ou garantir]
- [Ganho concreto, com número quando existir]

### Negativas

- [O que essa decisão custa. Latência adicional, código a manter, ponto único de falha,
  acoplamento criado, operação nova para o time]
- [Quando houver mitigação prevista, informe: "mitigado por ..." ou "endereçado em ADR-NNN"]

## Referências

- [Origem da decisão: `[hh:mm] Falante` para transcrição, caminho do arquivo para código]
- [ADRs relacionados]
- [RFC relacionado, quando existir]
```

## Notas sobre o preenchimento

**Status.** Uma decisão extraída de uma reunião já encerrada nasce `Aceito`. `Proposto` é
para decisão que ainda vai ser submetida. Um ADR nunca é apagado quando muda de ideia: ele
vira `Substituído por` e o novo traz `Substitui`.

**Contexto.** O erro mais comum é escrever o contexto já contaminado pela solução
("precisávamos de uma outbox"). O contexto correto descreve o problema de modo que as
alternativas ainda façam sentido ("precisávamos garantir que nenhum evento se perdesse se o
processo caísse entre a escrita no banco e a publicação").

**Decisão.** Se a frase precisa de "e" para ser escrita, verifique se não são dois ADRs.

**Alternativas.** Pelo menos uma, real. Se a fonte não registra alternativa nenhuma mas
existe uma óbvia no domínio, ela pode entrar marcada como `(alternativa plausível, não
discutida na fonte)`. O que não pode é uma alternativa inventada apresentada como se tivesse
sido debatida.

**Consequências negativas.** Obrigatórias. Toda decisão de arquitetura custa alguma coisa. Um
ADR sem custo declarado não foi analisado, foi anunciado.

## Índice (`README.md` da pasta de ADRs)

```markdown
# Architecture Decision Records

Registro das decisões arquiteturais [do projeto | da feature X]. Cada arquivo documenta uma
decisão, no formato MADR: contexto, decisão, alternativas consideradas e consequências.

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-....md) | [título] | Aceito |
| [ADR-002](ADR-002-....md) | [título] | Aceito |
```
