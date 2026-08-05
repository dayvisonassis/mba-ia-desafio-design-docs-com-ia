# Esqueleto do README de processo

Textos entre colchetes são instruções e não aparecem na saída.

```markdown
# [Nome da entrega]

## Sobre o desafio

[Um ou dois parágrafos, com suas palavras, descrevendo a tarefa: qual era o material de
entrada, o que precisava ser produzido e qual a restrição principal.]

---

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| [ferramenta] | [o que ela fez de fato nesta produção] |

[Quando skills, plugins ou agentes customizados foram usados, cite-os com o link do
repositório.]

---

## Workflow adotado

[Em que ordem os documentos foram produzidos e por quê.

Explique a lógica da ordem. Exemplo: decisões primeiro porque elas formam o esqueleto do
"como implementar"; PRD por último porque, com os demais prontos, ele vira consolidação.

Se a ordem real divergiu da planejada, conte a divergência e o motivo. É informação útil.]

```
[diagrama simples do fluxo, se ajudar]
```

---

## Prompts customizados

### [O que este prompt resolve]

[Uma linha sobre por que ele foi necessário e o que mudou no resultado.]

```
[o prompt, na íntegra, como foi usado]
```

### [O que este prompt resolve]

[Idem.]

```
[o prompt]
```

---

## Iterações e ajustes

[Os momentos em que a saída veio errada ou rasa e precisou de correção. Cada um com: o que a
IA produziu, por que estava errado, o que foi feito, e o que mudou no resultado.]

### 1. [Título curto do problema]

**O que veio:** [a saída original]

**Por que estava errado:** [o critério que ela violava]

**Correção:** [o que foi feito, incluindo o ajuste no prompt ou na skill]

**Resultado:** [o que mudou]

### 2. [Título curto do problema]

[Idem.]

---

## Como navegar a entrega

| Arquivo | O que contém |
| --- | --- |
| [`caminho`](caminho) | [conteúdo] |

**Ordem sugerida de leitura**

1. [`arquivo`](arquivo) — [por que começar por aqui]
2. [`arquivo`](arquivo) — [por que vem em seguida]
```

## Notas sobre o preenchimento

**Prompts.** Na íntegra, como foram usados, sem limpeza cosmética. Um prompt editado para
parecer melhor deixa de servir para reproduzir o processo, que é a razão de ele estar aqui.

**Iterações.** É a seção que separa um README honesto de um relato de marketing. O padrão
útil é sempre: *veio isto → estava errado por este critério → fiz aquilo → mudou isso*.
Genérico não serve. "A IA tratou como requisito uma ideia que a reunião tinha descartado, e
eu tive que criar uma etapa de classificação antes da geração" é concreto e ensina alguma
coisa.

**Ordem de leitura.** Siga as alturas dos documentos: por que e o quê, depois como propomos,
depois cada decisão, depois como construir. Quem lê na ordem certa entende o pacote sem
precisar voltar.
