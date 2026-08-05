# ADR-001: Padrão Outbox no MySQL para publicação de eventos de pedido

## Status

Aceito em 2026-08-05

## Contexto

Três clientes B2B pediram formalmente para serem notificados quando o status dos pedidos
deles muda na plataforma. Hoje eles fazem polling em `GET /orders`, o que deixa a integração
lenta e cara do lado deles. A tolerância declarada é de até 10 segundos entre a mudança e a
notificação.

A mudança de status acontece em `changeStatus`, em
`src/modules/orders/order.service.ts`, dentro de uma transação Prisma que já faz três coisas:
atualiza a `orders`, insere em `order_status_history` e movimenta `stockQuantity` dos
produtos do pedido quando a transição exige. A transação já é pesada.

Isso cria um problema de consistência entre dois sistemas. Se a mudança de status commitar e
a notificação não sair, o cliente nunca fica sabendo. Se a notificação sair e a transação der
rollback, o cliente é informado de um estado que não existe. O sistema precisa de uma
garantia de que evento registrado e status alterado são a mesma coisa, atomicamente.

A restrição de contexto que pesa sobre a escolha: o time é pequeno e não opera nenhum
componente de mensageria hoje. A stack atual é MySQL via Prisma, sem fila, sem broker, sem
cache distribuído.

## Decisão

Adotamos o padrão **Outbox**: quando o status de um pedido muda, uma linha de evento é
inserida em uma tabela `webhook_outbox` **dentro da mesma transação SQL** que atualiza
`orders` e `order_status_history`.

Isso significa que:

- se a transação commita, o evento está registrado;
- se a transação sofre rollback, o evento desaparece junto;
- não existe estado intermediário em que o status mudou e o evento não foi registrado.

A tabela vive no MySQL que já existe. Nenhum componente novo de infraestrutura é introduzido.

O identificador da linha segue o padrão do resto do projeto, UUID, como todos os modelos em
`prisma/schema.prisma`.

A tabela é indexada por status de processamento (pendente, processando, falhou, entregue) e
por data de criação, para que a leitura do worker toque apenas as linhas pendentes mais
antigas em lotes pequenos.

O consumo dessa tabela é decidido separadamente em
[ADR-002](ADR-002-worker-separado-em-polling.md).

## Alternativas Consideradas

### Disparo síncrono da chamada HTTP dentro de `changeStatus`

Emitir o webhook diretamente no service, no momento da mudança de status.

**Trade-off que levou ao descarte:** acopla a latência de um sistema externo à transação
principal. Um cliente lento seguraria a transação e, por consequência, travaria a mudança de
status de outros pedidos. Pior: com o cliente fora do ar, a única saída seria dar rollback em
uma mudança de status legítima, o que é inaceitável do ponto de vista do domínio.

### Redis Streams ou componente de mensageria dedicado

Publicar o evento em um stream e ter consumidores lendo dali.

**Trade-off que levou ao descarte:** exigiria subir e operar infraestrutura que o time não
tem hoje. Para o volume em questão, o MySQL existente resolve. Foi classificado na discussão
como overengineering para o tamanho do time.

## Consequências

### Positivas

- Consistência garantida por construção: evento e mudança de status commitam ou falham
  juntos, sem janela de inconsistência possível.
- Nenhuma dependência de infraestrutura nova. Mesmo banco, mesma stack, mesmo deploy.
- A tabela de outbox vira trilha de auditoria natural do que foi emitido e quando.
- A transação principal não fica exposta à latência ou à disponibilidade de sistemas
  externos.

### Negativas

- A transação de `changeStatus`, que já era pesada, ganha mais uma escrita. O impacto é
  pequeno em termos absolutos, mas é real e incide sobre o caminho crítico do domínio.
- Se a inserção na outbox falhar, a mudança de status inteira sofre rollback. É o
  comportamento desejado, mas significa que um defeito no módulo de webhooks passa a
  conseguir bloquear uma operação de negócio. Mitigado mantendo a lógica de inserção mínima:
  uma função que recebe o `tx` da transação corrente e apenas insere, sem chamada externa e
  sem regra complexa.
- A tabela cresce indefinidamente sem uma política de arquivamento. O arquivamento de linhas
  entregues foi explicitamente adiado para fora do escopo desta feature, o que deixa uma
  dívida operacional conhecida.
- A entrega passa a ser assíncrona, com latência mínima determinada pelo mecanismo de
  consumo, e não mais imediata.

## Referências

- Decisão fechada na reunião técnica de webhooks (`TRANSCRICAO.md`): `[09:06] Diego`
  apresenta o padrão, `[09:08] Larissa` fecha. A transcrição não registra a data da reunião;
  a data no Status é a do registro deste ADR.
- Alternativa síncrona descartada: `[09:04] Bruno`, `[09:06] Diego`
- Alternativa de mensageria descartada: `[09:07] Larissa`, `[09:07] Diego`
- Padrão de UUID: `[09:51] Larissa` e `prisma/schema.prisma`
- Transação afetada: `src/modules/orders/order.service.ts`
- [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md)
