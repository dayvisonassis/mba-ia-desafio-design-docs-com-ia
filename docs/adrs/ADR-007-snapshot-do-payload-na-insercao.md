# ADR-007: Snapshot do payload no momento da inserção na outbox

## Status

Aceito em 2026-08-05

## Contexto

A linha da outbox ([ADR-001](ADR-001-outbox-no-mysql.md)) é criada quando o status do pedido
muda, mas o envio acontece depois: no mínimo 2 segundos depois
([ADR-002](ADR-002-worker-separado-em-polling.md)) e, em cenário de retry, até cerca de 15
horas depois ([ADR-003](ADR-003-retry-com-backoff-e-dlq.md)).

Nesse intervalo o pedido pode mudar de novo. Um pedido que foi para `PAID` às 10h e para
`PROCESSING` às 10h05 tem, às 10h05, dois eventos na outbox e um estado atual que já não é o
de nenhum dos dois momentos.

A pergunta era o que a linha da outbox guarda: o payload já montado, ou apenas o `order_id`
com a montagem sendo feita na hora do envio.

## Decisão

**A linha da outbox guarda o payload já renderizado, montado no momento da inserção.** É um
snapshot do estado do pedido no instante em que a mudança de status ocorreu.

Consequência direta: o evento entregue ao cliente descreve o pedido como ele estava quando
aquela transição aconteceu, mesmo que o envio ocorra horas depois e o pedido já esteja em
outro estado.

O payload é montado em JSON e contém o identificador do evento, o tipo do evento
(`order.status_changed`), o timestamp em ISO 8601, o identificador e o número do pedido, o
status de origem e o de destino, o identificador do cliente e os campos básicos do pedido,
como o total. **Os itens do pedido não entram no payload**, para não inflá-lo; o cliente que
precisar do detalhe consulta `GET /orders/:id`.

## Alternativas Consideradas

### Guardar apenas o `order_id` e renderizar o payload na hora do envio

A linha da outbox seria mínima, e o worker consultaria o pedido no momento de montar a
requisição.

**Trade-off que levou ao descarte:** o payload passaria a refletir o estado do pedido no
momento do **envio**, não no momento do **evento**. Isso produz situações incoerentes: o
cliente receberia uma notificação dizendo que o pedido mudou de `PAID` para `PROCESSING`,
mas com os dados do pedido já em `SHIPPED`. Em cenário de retry, o descompasso pode ser de
horas. Um evento que se contradiz internamente é pior que um evento com dado antigo.

## Consequências

### Positivas

- O evento é internamente coerente: a transição descrita e os dados do pedido correspondem ao
  mesmo instante.
- O envio fica independente do estado atual do banco. O worker não precisa consultar o pedido
  para montar a requisição, o que simplifica o processamento e reduz consulta.
- Um evento na DLQ carrega o payload completo, o que torna a depuração possível meses depois,
  mesmo que o pedido tenha evoluído ou sido removido.
- O reprocessamento manual reenvia exatamente o que teria sido enviado na primeira tentativa,
  sem surpresa.

### Negativas

- Duplicação de dado: o conteúdo do pedido passa a existir também na outbox, e cresce com o
  volume de eventos. O custo é ampliado pela ausência de política de arquivamento, que foi
  adiada para fora do escopo desta feature.
- Um defeito na montagem do payload fica congelado na linha. Corrigir o código não corrige os
  eventos já inseridos e ainda não entregues; eles seriam reenviados com o payload errado.
- Se algum dado do pedido for corrigido depois da mudança de status, o cliente recebe a versão
  antiga e não fica sabendo da correção, porque uma correção de dado não gera transição de
  status e, portanto, não gera evento.
- O limite de 64KB de payload passa a ser avaliado na inserção, dentro da transação de
  `changeStatus`, e não no envio. Um evento acima do limite falha em um ponto mais sensível
  do sistema.

## Referências

- Pergunta levantada e decisão: `[09:51] Bruno`, `[09:52] Larissa`, confirmada em
  `[09:52] Diego` e `[09:52] Bruno`
- Formato do payload: `[09:43] Diego`
- Exclusão dos itens do pedido: `[09:43] Diego`, `[09:44] Bruno`
- Limite de 64KB: `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa`
- Arquivamento fora de escopo: `[09:08] Diego`
- [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md)
