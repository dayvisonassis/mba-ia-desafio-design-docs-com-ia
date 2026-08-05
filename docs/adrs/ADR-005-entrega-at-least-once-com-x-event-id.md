# ADR-005: Entrega at-least-once com deduplicação pelo cliente via X-Event-Id

## Status

Aceito em 2026-08-05

## Contexto

A combinação de outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) com worker e retry
([ADR-002](ADR-002-worker-separado-em-polling.md),
[ADR-003](ADR-003-retry-com-backoff-e-dlq.md)) cria cenários em que o mesmo evento pode ser
entregue mais de uma vez.

O caso clássico: o worker envia a requisição, o cliente processa e responde com sucesso, mas
a resposta se perde ou o worker morre antes de marcar a linha como entregue. No próximo
ciclo, o evento volta a aparecer como pendente e é enviado de novo. Do ponto de vista do
worker, nada distingue "não foi entregue" de "foi entregue mas eu não soube".

Era preciso decidir qual garantia de entrega o sistema oferece e quem paga o custo dela.

## Decisão

O sistema garante **at-least-once**: todo evento é entregue pelo menos uma vez, e a entrega
duplicada é um comportamento possível e documentado, não um defeito.

Para tornar a duplicata tratável, cada evento carrega um **identificador único**, gerado no
momento em que ele entra na outbox e enviado no header **`X-Event-Id`**. O identificador é
estável: as retentativas do mesmo evento reenviam o mesmo `X-Event-Id`.

A **deduplicação é responsabilidade do cliente**, que registra os identificadores já
processados e ignora repetições. Isso é comunicado de forma destacada na documentação de
integração.

## Alternativas Consideradas

### Exactly-once

Garantir que cada evento chegue exatamente uma vez.

**Trade-off que levou ao descarte:** exactly-once real exige coordenação entre os dois lados,
com confirmação transacional do processamento por parte do cliente. Isso transformaria uma
integração simples por HTTP em um protocolo com estado compartilhado, aumentando muito a
complexidade dos dois lados. At-least-once com identificador de evento resolve
essencialmente os mesmos casos práticos, e é o que Stripe e GitHub fazem, o que significa que
a maioria dos clientes já tem o padrão implementado.

### Deduplicação do lado da plataforma

A plataforma manteria o registro do que já foi confirmado e evitaria reenviar.

**Trade-off que levou ao descarte:** não resolve o problema de raiz. A ambiguidade está
exatamente no caso em que a plataforma não sabe se a entrega chegou. Guardar mais estado do
lado de cá reduz a probabilidade de duplicata, mas não a elimina, e ainda assim o cliente
precisaria estar preparado para recebê-la. Investir na garantia do lado errado da fronteira
adiciona complexidade sem remover a necessidade do tratamento.

## Consequências

### Positivas

- Nenhum evento é perdido, que é a garantia que realmente importa para o caso de uso.
- O mecanismo é simples e não exige coordenação entre os dois lados. Um header e um
  identificador.
- Está alinhado ao padrão de mercado, o que reduz a fricção de integração: clientes que já
  consomem webhooks de outros provedores reconhecem o modelo.
- O identificador serve também como chave de correlação para depuração ponta a ponta,
  ligando a linha da outbox, o log do envio e o registro de entrega.

### Negativas

- Transfere responsabilidade para o cliente. Um cliente que ignore o `X-Event-Id` vai
  processar duplicatas, e o efeito disso depende do que ele faz com o evento. Se a operação
  dele não for idempotente, a duplicata vira erro no sistema dele.
- A plataforma não tem como verificar se o cliente realmente dedupica. O problema só aparece
  quando já causou dano.
- Aumenta o peso da documentação de integração: o comportamento precisa ser comunicado de
  forma destacada, e ainda assim haverá quem não leia.
- Combinado com o retry longo ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)), a duplicata
  pode chegar horas depois da primeira entrega, quando o cliente já pode ter descartado o
  registro do identificador se a janela de deduplicação dele for curta.

## Referências

- Garantia at-least-once: `[09:24] Diego`
- `X-Event-Id` com UUID gerado na entrada da outbox: `[09:25] Diego`
- Objeção sobre transferir responsabilidade: `[09:25] Sofia`
- Justificativa contra exactly-once e referência ao padrão de mercado: `[09:25] Diego`
- Compromisso de documentar no portal do desenvolvedor: `[09:26] Marcos`
- Decisão fechada: `[09:26] Larissa`
- [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)
