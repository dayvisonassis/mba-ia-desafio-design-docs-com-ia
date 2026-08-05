# ADR-003: Retry com backoff exponencial e dead letter queue em tabela separada

## Status

Aceito em 2026-08-05

## Contexto

O worker ([ADR-002](ADR-002-worker-separado-em-polling.md)) entrega eventos para endpoints
HTTP que estão fora da infraestrutura da plataforma. Esses endpoints saem do ar. Já houve
caso de cliente com indisponibilidade de duas horas em manutenção planejada.

Uma falha de entrega não pode simplesmente descartar o evento: o cliente pediu a notificação
justamente para não precisar fazer polling, e um evento perdido é exatamente o que o padrão
Outbox foi adotado para evitar. Por outro lado, retentar para sempre também não serve: se um
cliente sumiu de vez, o evento fica pendurado indefinidamente e a fila nunca drena.

Era preciso decidir três coisas amarradas: quantas vezes retentar, com que espaçamento, e o
que fazer com o que esgota as tentativas.

> Este ADR registra retry e DLQ juntos porque as duas decisões são inseparáveis: a política
> de retry só está completa quando define o destino do evento que esgota as tentativas, e a
> DLQ só existe porque o retry tem um teto. Separá-las produziria dois registros que só fazem
> sentido lidos em conjunto.

## Decisão

**Retry com backoff exponencial, 5 tentativas, com a progressão 1 minuto, 5 minutos, 30
minutos, 2 horas, 12 horas.** A janela total entre a primeira falha e a última tentativa é de
aproximadamente 15 horas.

Uma tentativa é considerada falha quando a resposta não é de sucesso ou quando o timeout de
10 segundos é atingido.

**Esgotadas as 5 tentativas, o evento é movido para uma tabela dedicada,
`webhook_dead_letter`**, persistindo o payload, o motivo da última falha e o timestamp. A
outbox principal não guarda o estado terminal de falha.

**Reprocessamento é manual, via endpoint administrativo**
(`POST /admin/webhooks/dead-letter/:id/replay`), que recoloca o evento na outbox como
pendente. O controle de acesso desse endpoint está em
[ADR-006](ADR-006-reuso-dos-padroes-existentes.md).

## Alternativas Consideradas

### 3 tentativas em vez de 5

Uma política mais agressiva, falhando rápido.

**Trade-off que levou ao descarte:** com backoff exponencial, três tentativas cobrem uma
janela de aproximadamente 30 minutos. Um cliente que teve indisponibilidade pela manhã
perderia os eventos, e já houve cliente com duas horas de manutenção planejada. O ganho em
drenar a fila mais rápido não compensa a perda de eventos em cenários que comprovadamente
acontecem.

### Retry indefinido com backoff

Nunca desistir, apenas aumentar o intervalo entre tentativas.

**Trade-off que levou ao descarte:** um cliente que desaparece deixa eventos pendurados para
sempre, e a fila acumula trabalho que nunca vai concluir. Sem um estado terminal não existe
sinal claro de que a integração está quebrada, nem momento definido para intervenção humana.

### Marcar o evento como `failed` na própria outbox, sem tabela separada

Manter tudo em uma tabela só, distinguindo pelo status.

**Trade-off que levou ao descarte:** polui a leitura da outbox principal, que é consultada a
cada 2 segundos pelo worker, com linhas que nunca mais serão processadas. A tabela separada
mantém a outbox enxuta e funciona como evidência isolada para depuração e reprocessamento.

## Consequências

### Positivas

- Cobre indisponibilidades reais de cliente, incluindo manutenções longas, sem intervenção
  humana.
- O backoff crescente evita martelar um endpoint que já está em dificuldade.
- Existe um estado terminal explícito. Um evento na DLQ é um sinal inequívoco de que aquela
  integração precisa de atenção.
- A outbox principal permanece enxuta, e a consulta do worker continua barata.
- A DLQ preserva payload e motivo da falha, o que torna a depuração possível depois do fato.
- O replay manual permite recuperar eventos após a correção do problema do lado do cliente.

### Negativas

- Um evento pode levar até cerca de 15 horas para chegar ao cliente em cenário de falha
  prolongada. É muito acima dos 10 segundos do caminho feliz, e o cliente precisa estar
  ciente disso.
- Cinco tentativas espaçadas significam que uma linha da outbox pode permanecer ativa por
  quase um dia, ocupando espaço e aparecendo nas consultas.
- O reprocessamento é manual. Se ninguém olhar a DLQ, os eventos ficam lá indefinidamente. A
  decisão de não notificar automaticamente o cliente sobre falhas recorrentes por email foi
  adiada para uma fase futura, o que deixa a descoberta dependente de monitoramento interno.
- Duas tabelas para modelar, manter e consultar em vez de uma.
- O evento entregue com 12 horas de atraso chega fora de ordem em relação a eventos mais
  recentes do mesmo pedido, quebrando na prática a ordenação que o worker único garante no
  caminho feliz.

## Referências

- Backoff exponencial e DLQ: `[09:15] Diego`
- Cinco tentativas, com a justificativa contra três: `[09:15] Diego`, `[09:16] Bruno`,
  `[09:16] Diego`
- Progressão do backoff: `[09:17] Diego`, aceita em `[09:17] Marcos`, fechada em
  `[09:17] Larissa`
- Tabela `webhook_dead_letter` separada: `[09:18] Diego`
- Endpoint de replay: `[09:18] Diego`, `[09:35] Diego`
- Timeout de 10 segundos: `[09:42] Diego`
- Email de alerta adiado: `[09:37] Larissa`
- [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)
