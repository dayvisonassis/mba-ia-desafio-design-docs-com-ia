# ADR-002: Worker em processo separado consumindo a outbox por polling

## Status

Aceito em 2026-08-05

## Contexto

Com a adoção do padrão Outbox ([ADR-001](ADR-001-outbox-no-mysql.md)), os eventos passam a
ser gravados em uma tabela e alguém precisa consumi-los e disparar as chamadas HTTP para os
endpoints dos clientes.

Duas perguntas precisavam de resposta: **como** esse consumidor descobre que há evento novo,
e **onde** ele roda.

Sobre o "como", o banco é MySQL. Diferente do PostgreSQL, o MySQL não tem mecanismo nativo de
notificação para processo externo: não existe equivalente ao `LISTEN`/`NOTIFY`. Triggers
existem, mas executam SQL, não avisam aplicação.

Sobre o "onde", a API hoje sobe por `src/server.ts`, que carrega o app Express definido em
`src/app.ts`. Colocar o consumidor dentro desse mesmo processo o amarraria ao ciclo de vida da
API.

A tolerância de latência acordada com os clientes é de até 10 segundos.

## Decisão

O consumidor da outbox é um **worker em processo separado**, que descobre eventos novos por
**polling a cada 2 segundos**.

Concretamente:

- Entry-point nova, a ser criada em `src/worker.ts`, no mesmo nível do `src/server.ts`
  existente, acionada por um script dedicado (`npm run worker`).
- Mesmo banco, mesma `DATABASE_URL`, mesma stack. O que não pode ser compartilhado é o
  processo.
- Instância própria de `PrismaClient`, porque o client é por processo.
- A cada ciclo, o worker busca os eventos pendentes mais antigos em lote pequeno, processa e
  marca o resultado.
- A lógica de processamento fica dentro do módulo, em arquivo a ser criado no caminho
  `src/modules/webhooks/webhook.worker.ts`, seguindo a organização por módulo já adotada no
  projeto ([ADR-006](ADR-006-reuso-dos-padroes-existentes.md)).

**Ordenação.** Com um único worker, o processamento segue a ordem de criação das linhas da
outbox e a entrega chega ordenada ao cliente. Essa garantia é aceita explicitamente como
**limitação conhecida**: ela vale por `order_id` e apenas enquanto houver um único worker. Não
há garantia de ordenação global, e os clientes não pediram uma.

## Alternativas Consideradas

### Trigger de banco notificando o worker

Usar trigger no MySQL para reagir à inserção na outbox e acordar o consumidor.

**Trade-off que levou ao descarte:** o MySQL não tem listener nativo para processo externo.
Uma trigger só executa SQL. Para avisar o worker, seria preciso improvisar algo como escrever
em arquivo ou fazer a trigger bater em um endpoint HTTP, o que acrescenta um mecanismo frágil
e não convencional para ganhar poucos segundos de latência que o requisito não exige.

### Worker dentro do mesmo processo da API

Rodar o loop de consumo dentro da instância que serve o Express.

**Trade-off que levou ao descarte:** amarra o consumo ao ciclo de vida da API. Um restart ou
um deploy da API derruba o worker junto, e a entrega de notificações passa a depender da
saúde do processo HTTP. São responsabilidades com perfis de disponibilidade diferentes e
merecem processos diferentes.

### Múltiplos workers em paralelo desde o início

Escalar horizontalmente o consumo já na primeira versão.

**Trade-off que levou ao descarte:** perde a ordenação implícita por `order_id` sem trazer
benefício para o volume atual. Manter um único worker preserva a ordenação de graça. Quando a
escala exigir, as saídas conhecidas são particionar por `order_id` ou usar lock pessimista,
e a decisão será tomada então.

## Consequências

### Positivas

- Latência máxima de 2 segundos entre o commit da mudança de status e o início do envio,
  bem dentro dos 10 segundos acordados.
- Mecanismo simples, previsível e fácil de depurar. Um `SELECT` com filtro e ordenação, sem
  infraestrutura de notificação.
- Isolamento de falhas: a API não cai porque o worker travou, e o worker não morre porque a
  API reiniciou.
- Ordenação por `order_id` sai de graça enquanto o worker for único.
- Reinício do worker é seguro: o estado de progresso vive no banco, não em memória.

### Negativas

- Latência mínima de até 2 segundos mesmo quando o sistema está ocioso. É um piso, não uma
  média, e não há como baixá-lo sem aumentar a frequência do polling.
- Consulta recorrente ao banco a cada 2 segundos, mesmo sem evento algum para processar.
  Mitigado pelos índices em status de processamento e data de criação, que mantêm a consulta
  barata.
- Um novo processo para operar, monitorar e implantar. Se o worker cair e ninguém perceber,
  os eventos se acumulam silenciosamente na outbox sem que nenhuma requisição HTTP falhe.
  Isso torna o monitoramento do próprio worker um requisito, não um extra.
- O worker é ponto único de processamento. Não há failover: enquanto ele estiver fora, nada
  é entregue.
- A ordenação garantida hoje vira uma expectativa dos clientes que a evolução para múltiplos
  workers vai quebrar.

## Referências

- Decisão de polling: `[09:09] Diego`, fechada em `[09:10] Larissa`
- Trigger descartada: `[09:09] Bruno` pergunta, `[09:09] Diego` explica a limitação do MySQL
- Processo separado: `[09:11] Diego`, `[09:11] Larissa` propõe `src/worker.ts`
- `PrismaClient` por processo: `[09:30] Bruno`
- Ordenação como limitação conhecida: `[09:12] Diego`, `[09:13] Larissa`, `[09:14] Marcos`
- Entry-point atual: `src/server.ts`, `src/app.ts`
- [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-com-backoff-e-dlq.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)
