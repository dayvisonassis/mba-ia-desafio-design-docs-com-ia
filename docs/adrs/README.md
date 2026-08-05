# Architecture Decision Records

Registro das decisões arquiteturais do Sistema de Webhooks de Notificação de Pedidos. Cada
arquivo documenta uma decisão no formato MADR: contexto, decisão, alternativas consideradas e
consequências positivas e negativas.

Todas as decisões foram fechadas na reunião técnica registrada em
[`TRANSCRICAO.md`](../../TRANSCRICAO.md). A seção **Referências** de cada ADR aponta o
timestamp e o participante que originou cada ponto.

| ADR | Decisão | Status |
| --- | --- | --- |
| [ADR-001](ADR-001-outbox-no-mysql.md) | Padrão Outbox no MySQL para publicação de eventos de pedido | Aceito |
| [ADR-002](ADR-002-worker-separado-em-polling.md) | Worker em processo separado consumindo a outbox por polling | Aceito |
| [ADR-003](ADR-003-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial e dead letter queue em tabela separada | Aceito |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | Autenticação por HMAC-SHA256 com secret única por endpoint e rotação com grace period | Aceito |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação pelo cliente via X-Event-Id | Aceito |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões existentes do projeto no módulo de webhooks | Aceito |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload no momento da inserção na outbox | Aceito |

## Como estes ADRs se relacionam

```
ADR-001  Outbox no MySQL
   │
   ├─► ADR-002  Worker separado em polling      (como a outbox é consumida)
   │      │
   │      └─► ADR-003  Retry com backoff e DLQ  (o que acontece quando a entrega falha)
   │             │
   │             └─► ADR-005  At-least-once com X-Event-Id  (consequência do retry)
   │
   └─► ADR-007  Snapshot do payload             (o que a linha da outbox guarda)

ADR-004  HMAC-SHA256 com secret por endpoint    (como o cliente confia no que recebe)

ADR-006  Reuso dos padrões existentes           (transversal a todas as anteriores)
```

Decisões técnicas secundárias que não viraram ADR próprio, por não terem peso arquitetural,
estão detalhadas no [FDD](../FDD.md): formato exato do payload, conjunto de headers,
timeout de 10 segundos, limite de 64KB e validação de URL `https`.
