# FDD: Sistema de Webhooks de Notificação de Pedidos

| | |
| --- | --- |
| **Versão** | 1.0 |
| **Data** | 2026-08-05 |
| **Responsável técnico** | Larissa (Tech Lead) |
| **Documentos relacionados** | [PRD](PRD.md), [RFC](RFC.md), [ADRs](adrs/README.md) |

> **Convenção de caminhos.** Este documento cita arquivos existentes do repositório e arquivos
> a serem criados. Os que devem ser criados estão sempre marcados como **(novo)**. Todos os
> demais existem hoje e podem ser abertos.

---

## 1. Contexto e motivação técnica

O Order Management System não possui hoje nenhum mecanismo de notificação externa, evento ou
fila. Toda a comunicação é síncrona, por requisição HTTP de entrada. Clientes B2B que precisam
saber de mudanças de status recorrem a polling em `GET /api/v1/orders`.

Esta feature preenche esse vazio para um caso específico: a transição de status de pedido. A
motivação de negócio está no [PRD](PRD.md); a abordagem e as alternativas descartadas estão no
[RFC](RFC.md); cada decisão isolada está nos [ADRs](adrs/README.md). Este documento detalha
como construir.

O problema técnico central é de **consistência entre dois sistemas**. A mudança de status
acontece em `changeStatus`, em `src/modules/orders/order.service.ts`, dentro de uma transação
Prisma. A notificação precisa acontecer se e somente se essa transação commitar, e não pode
depender da disponibilidade do destinatário. A solução é o padrão Outbox
([ADR-001](adrs/ADR-001-outbox-no-mysql.md)), com entrega assíncrona por um worker em processo
separado ([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)).

**Atores**

| Ator | Papel |
| --- | --- |
| API (processo existente) | Recebe o CRUD de configuração e insere eventos na outbox durante `changeStatus` |
| Worker (processo novo) | Lê a outbox, assina e envia as requisições, aplica retry e alimenta a DLQ |
| Endpoint do cliente | Recebe as notificações e valida a assinatura |
| Operador ADMIN | Reprocessa eventos da dead letter queue |

**Suposições e restrições**

- MySQL, acessado via Prisma 5.22. Sem broker, sem cache distribuído, sem fila.
- Node 20 ou superior, o que dá acesso ao `fetch` nativo, ao `AbortSignal.timeout` e ao módulo
  `node:crypto` sem dependência nova.
- O worker roda como instância única. O paralelismo está fora desta entrega.
- Um mesmo cliente pode ter mais de um endpoint cadastrado.

---

## 2. Objetivos técnicos

- **Nenhum evento perdido.** Se `changeStatus` commitou e havia endpoint interessado no status
  de destino, existe uma linha correspondente em `webhook_outbox`. Invariante verificável por
  teste de integração com falha injetada.
- **Nenhum evento fantasma.** Se a transação sofreu rollback, não existe linha de evento.
- **Latência de primeira tentativa abaixo de 10 segundos** no caminho feliz, com piso de 2
  segundos determinado pelo intervalo de polling.
- **Entrega at-least-once com identificador estável.** Todas as tentativas do mesmo evento
  carregam o mesmo `X-Event-Id`.
- **Toda requisição enviada é verificável** pelo destinatário por HMAC-SHA256.
- **Nenhum evento descartado em silêncio.** Um evento que esgota as tentativas está em
  `webhook_dead_letter` com o motivo da última falha.
- **Zero alteração de contrato existente.** Os endpoints atuais da API respondem exatamente
  como respondem hoje.

---

## 3. Escopo e exclusões

**Incluído**

- Modelagem de `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e
  `webhook_dead_letter`
- Inserção transacional do evento durante `changeStatus`, com filtro por status na inserção
- Worker de entrega com polling, retry, backoff e movimentação para DLQ
- Assinatura HMAC-SHA256 e rotação de secret com grace period
- CRUD de configuração de webhook e consulta de histórico de entregas
- Endpoint administrativo de replay de dead letter

**Excluído**

- **Notificação por email** ao cliente com webhook falhando. Adiado para fase futura, depois
  da medição de impacto.
- **Dashboard visual** para o cliente. Projeto separado do time de frontend.
- **Arquivamento das linhas entregues** da outbox. Fora do escopo desta feature.
- **Webhooks de entrada** (cliente para plataforma). O escopo é outbound apenas.
- **Rate limiting de envio** por cliente. Questão em aberto, a observar em produção.
- **Múltiplos workers em paralelo.** A ordenação atual depende de o worker ser único.

---

## 4. Fluxos detalhados

### 4.1 Criação do evento na outbox

Ocorre dentro da transação existente de `changeStatus`, em
`src/modules/orders/order.service.ts`.

1. A transação valida a transição com `canTransition` (`src/modules/orders/order.status.ts`) e
   movimenta estoque quando a transição exige.
2. A transação atualiza `orders` e insere em `order_status_history`, como hoje.
3. **Passo novo**, ainda dentro do mesmo `tx`: chamada a
   `publishWebhookEvent(tx, order, fromStatus, toStatus)`, uma função que recebe o client da
   transação corrente em vez de um repository injetado.
4. A função busca os endpoints **ativos** do `customerId` do pedido cujo filtro de eventos
   inclui o status de destino.
5. **Se nenhum endpoint corresponder, nada é inserido** e a função retorna. Esse é o filtro na
   inserção: se ninguém quer o status, a linha não existe.
6. Para cada endpoint correspondente, a função monta o payload completo (snapshot,
   [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)) e insere **uma linha** em
   `webhook_outbox`, com `status = PENDING`, `attempts = 0` e `nextRetryAt = now()`.
7. Se o payload serializado ultrapassar 64KB, a função lança `WEBHOOK_PAYLOAD_TOO_LARGE`, o
   que provoca rollback da transação inteira.
8. A transação commita. O evento existe.

> **Uma linha por par (evento, endpoint).** O estado de retry é por destinatário: se um
> cliente tem dois endpoints e um deles falha, apenas aquele retenta. Como consequência, cada
> linha tem seu próprio `X-Event-Id`, o que também evita que a deduplicação do cliente descarte
> a entrega destinada ao segundo endpoint.

### 4.2 Processamento pelo worker

Loop contínuo em processo separado, iniciado por `src/worker.ts` **(novo)**.

1. A cada 2 segundos, o worker seleciona um lote de linhas com `status = PENDING` e
   `nextRetryAt <= now()`, ordenado por `createdAt` ascendente.
2. Marca as linhas selecionadas como `PROCESSING`, para que um reinício do worker no meio do
   ciclo não perca a referência do que estava em voo.
3. Para cada linha:
   1. Carrega o endpoint de destino e verifica se continua ativo. Endpoint inativo faz a linha
      ir direto para `DELIVERED` com resultado `SKIPPED`, sem tentativa de envio.
   2. Calcula a assinatura HMAC-SHA256 sobre o corpo exato que será enviado, usando a secret
      vigente do endpoint.
   3. Monta os headers e faz a requisição `POST` para a URL do endpoint, com timeout de 10
      segundos.
   4. Registra uma linha em `webhook_deliveries` com o número da tentativa, o status de
      resposta, um trecho do corpo da resposta e a duração.
   5. **Resposta 2xx:** marca a linha da outbox como `DELIVERED` e segue.
   6. **Resposta não 2xx, timeout ou erro de rede:** vai para o fluxo de retry.
4. Encerrado o lote, o worker aguarda o próximo ciclo.

### 4.3 Retry

1. Incrementa `attempts` na linha da outbox e persiste `lastError`.
2. Se `attempts < 5`, calcula `nextRetryAt = now() + backoff[attempts]`, onde `backoff` é a
   progressão fixa `[1min, 5min, 30min, 2h, 12h]`, e devolve a linha para `PENDING`.
3. Se `attempts >= 5`, segue para o fluxo de dead letter.

A progressão é uma tabela fixa, não uma fórmula calculada. A janela total entre a primeira
falha e a última tentativa é de aproximadamente 15 horas
([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).

### 4.4 Dead letter e replay

**Entrada na DLQ**

1. Insere em `webhook_dead_letter` uma linha com o payload, o `endpointId`, o número de
   tentativas, o motivo da última falha e o timestamp.
2. Marca a linha da outbox como `FAILED`.
3. Emite log em nível `error` e incrementa a métrica de entrada em DLQ.

**Replay manual**

1. Um operador com role `ADMIN` chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`.
2. O sistema recria uma linha em `webhook_outbox` com o mesmo payload e o mesmo `X-Event-Id`,
   com `status = PENDING` e `attempts = 0`.
3. Marca a linha da DLQ com `replayedAt` e o identificador de quem executou, para auditoria.
4. O worker processa no ciclo seguinte, como qualquer evento pendente.

O `X-Event-Id` é preservado no replay. Do ponto de vista do cliente, é uma nova tentativa do
mesmo evento, e a deduplicação dele continua funcionando.

### 4.5 Rotação de secret

1. O cliente chama `POST /api/v1/webhooks/:id/rotate-secret`.
2. O sistema gera uma nova secret, move a atual para `previousSecret` e define
   `previousSecretExpiresAt = now() + 24h`.
3. A nova secret é retornada **uma única vez**, no corpo da resposta.
4. A partir daí, a plataforma assina com a nova secret. A anterior permanece registrada apenas
   para que o cliente saiba qual ainda aceitar durante a janela.
5. Uma nova rotação dentro da janela é rejeitada com `WEBHOOK_ROTATION_IN_PROGRESS`, para não
   invalidar uma secret que o cliente ainda pode estar usando.

---

## 5. Contratos públicos

Base: `/api/v1`, montada em `src/app.ts`. Todos os endpoints exigem o header
`Authorization: Bearer <jwt>`, aplicado pelo `authenticate` de
`src/middlewares/auth.middleware.ts`.

### 5.1 `POST /api/v1/webhooks`

Cadastra um endpoint de webhook. A secret é gerada pela plataforma e devolvida **apenas nesta
resposta**.

**Request**

```json
{
  "customerId": "9b2f1c34-5a7e-4f21-8d90-3c6b1a2e7f45",
  "url": "https://api.atlascomercial.com.br/integracoes/oms/webhook",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true
}
```

| Campo | Tipo | Obrigatório | Semântica |
| --- | --- | --- | --- |
| `customerId` | uuid | sim | Cliente dono do endpoint. Vem do corpo, não do JWT: o JWT é do usuário operador, não do cliente |
| `url` | string | sim | Destino da notificação. **Precisa ser `https`** |
| `events` | array de `OrderStatus` | sim | Status que este endpoint quer receber. Valores válidos: `PENDING`, `PAID`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED` |
| `active` | boolean | não | Padrão `true` |

**Response `201 Created`**

```json
{
  "id": "3f8a12b7-90cd-4e55-b1a2-77e4d6c05913",
  "customerId": "9b2f1c34-5a7e-4f21-8d90-3c6b1a2e7f45",
  "url": "https://api.atlascomercial.com.br/integracoes/oms/webhook",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_7d4f9a1c8e2b6035a9f1d7c4e8b30a26",
  "createdAt": "2026-08-05T14:22:10.431Z"
}
```

> `secret` aparece somente aqui e na resposta de rotação. Nas demais leituras o campo é
> omitido.

| Código | Significado |
| --- | --- |
| `201` | Endpoint criado |
| `400` | Corpo inválido (`VALIDATION_ERROR`) ou URL não `https` (`WEBHOOK_INVALID_URL`) |
| `400` | Status inexistente na lista de eventos (`WEBHOOK_INVALID_STATUS_FILTER`) |
| `401` | Sem token ou token inválido |
| `404` | `customerId` não existe (`NOT_FOUND`) |

### 5.2 `GET /api/v1/webhooks?customerId=<uuid>`

Lista os endpoints de um cliente. A secret nunca é retornada.

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "3f8a12b7-90cd-4e55-b1a2-77e4d6c05913",
      "customerId": "9b2f1c34-5a7e-4f21-8d90-3c6b1a2e7f45",
      "url": "https://api.atlascomercial.com.br/integracoes/oms/webhook",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "secretRotatedAt": null,
      "createdAt": "2026-08-05T14:22:10.431Z"
    }
  ],
  "page": 1,
  "pageSize": 20,
  "total": 1
}
```

O envelope de paginação segue `paginated` de `src/shared/http/response.ts`.

| Código | Significado |
| --- | --- |
| `200` | Lista retornada, possivelmente vazia |
| `400` | `customerId` ausente ou não é uuid |
| `401` | Sem token ou token inválido |

### 5.3 `PATCH /api/v1/webhooks/:id`

Edita URL, filtro de eventos ou estado ativo. Campos ausentes permanecem inalterados. A secret
não é alterada por aqui.

**Request**

```json
{
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false
}
```

**Response `200 OK`**

```json
{
  "id": "3f8a12b7-90cd-4e55-b1a2-77e4d6c05913",
  "customerId": "9b2f1c34-5a7e-4f21-8d90-3c6b1a2e7f45",
  "url": "https://api.atlascomercial.com.br/integracoes/oms/webhook",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false,
  "updatedAt": "2026-08-05T15:03:44.008Z"
}
```

| Código | Significado |
| --- | --- |
| `200` | Endpoint atualizado |
| `400` | Corpo inválido, URL não `https` ou status inexistente |
| `401` | Sem token ou token inválido |
| `404` | Endpoint não existe (`WEBHOOK_NOT_FOUND`) |

Desativar um endpoint não cancela os eventos já pendentes na outbox para ele: o worker os
encerra como `SKIPPED` no momento do processamento.

### 5.4 `DELETE /api/v1/webhooks/:id`

Remove a configuração do endpoint.

**Response `204 No Content`** (sem corpo)

| Código | Significado |
| --- | --- |
| `204` | Endpoint removido |
| `401` | Sem token ou token inválido |
| `404` | Endpoint não existe (`WEBHOOK_NOT_FOUND`) |

O histórico em `webhook_deliveries` é preservado para auditoria; os eventos ainda pendentes na
outbox para o endpoint removido são encerrados como `SKIPPED`.

### 5.5 `POST /api/v1/webhooks/:id/rotate-secret`

Gera uma nova secret e mantém a anterior válida por 24 horas
([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)).

**Request:** sem corpo.

**Response `200 OK`**

```json
{
  "id": "3f8a12b7-90cd-4e55-b1a2-77e4d6c05913",
  "secret": "whsec_2a90c7e5b4136f8d0e2a95c71b4d8f30",
  "previousSecretExpiresAt": "2026-08-06T15:10:00.000Z",
  "rotatedAt": "2026-08-05T15:10:00.000Z"
}
```

| Código | Significado |
| --- | --- |
| `200` | Secret rotacionada. A nova secret é exibida uma única vez |
| `401` | Sem token ou token inválido |
| `404` | Endpoint não existe (`WEBHOOK_NOT_FOUND`) |
| `409` | Já existe rotação em janela de graça (`WEBHOOK_ROTATION_IN_PROGRESS`) |

### 5.6 `GET /api/v1/webhooks/:id/deliveries`

Histórico de entregas do endpoint, mais recentes primeiro. Padrão de 100 registros por página.

**Response `200 OK`**

```json
{
  "data": [
    {
      "id": "c1d5e920-7b34-4a86-9f02-5e71b8d3a604",
      "eventId": "a7f3c811-24d9-4b60-8e15-9c02f7ab5d38",
      "eventType": "order.status_changed",
      "attempt": 2,
      "result": "SUCCESS",
      "responseStatus": 200,
      "durationMs": 342,
      "attemptedAt": "2026-08-05T15:12:07.882Z",
      "payload": {
        "event_id": "a7f3c811-24d9-4b60-8e15-9c02f7ab5d38",
        "event_type": "order.status_changed",
        "timestamp": "2026-08-05T15:06:59.120Z",
        "order_id": "5c8e0b74-1a92-4d3f-b6e7-08a2c4f19d53",
        "order_number": "ORD-000482",
        "from_status": "PROCESSING",
        "to_status": "SHIPPED",
        "customer_id": "9b2f1c34-5a7e-4f21-8d90-3c6b1a2e7f45",
        "total_cents": 148900
      }
    },
    {
      "id": "b0a41f78-33c2-4de9-91b5-2f6c8e07a145",
      "eventId": "a7f3c811-24d9-4b60-8e15-9c02f7ab5d38",
      "eventType": "order.status_changed",
      "attempt": 1,
      "result": "FAILURE",
      "responseStatus": 503,
      "durationMs": 10000,
      "error": "WEBHOOK_DELIVERY_TIMEOUT",
      "attemptedAt": "2026-08-05T15:07:01.004Z"
    }
  ],
  "page": 1,
  "pageSize": 100,
  "total": 2
}
```

| Código | Significado |
| --- | --- |
| `200` | Histórico retornado |
| `401` | Sem token ou token inválido |
| `404` | Endpoint não existe (`WEBHOOK_NOT_FOUND`) |

### 5.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay`

Recoloca um evento da dead letter queue na outbox. **Exige role `ADMIN`**, aplicada pelo
`requireRole` de `src/middlewares/auth.middleware.ts`.

**Request:** sem corpo.

**Response `202 Accepted`**

```json
{
  "deadLetterId": "e4b7a015-6c39-42d8-b0f1-7a35c9e2408b",
  "outboxEventId": "a7f3c811-24d9-4b60-8e15-9c02f7ab5d38",
  "status": "PENDING",
  "replayedAt": "2026-08-05T16:40:12.559Z",
  "replayedBy": "1d0c93a6-8b52-4e17-a3f9-64d28e5c07b1"
}
```

| Código | Significado |
| --- | --- |
| `202` | Evento recolocado na outbox. A entrega ocorre no próximo ciclo do worker |
| `401` | Sem token ou token inválido |
| `403` | Role diferente de `ADMIN` (`FORBIDDEN`) |
| `404` | Registro não existe na DLQ (`WEBHOOK_DEAD_LETTER_NOT_FOUND`) |
| `409` | Registro já reprocessado (`WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED`) |

### 5.8 Contrato de saída: a requisição enviada ao cliente

Este é o contrato que o cliente implementa do lado dele.

`POST <url configurada>`

**Headers**

| Header | Semântica |
| --- | --- |
| `Content-Type` | Sempre `application/json` |
| `X-Event-Id` | UUID do evento. **Estável entre retentativas.** Chave de deduplicação do cliente |
| `X-Webhook-Id` | Identificador do endpoint cadastrado. Permite ao cliente com vários cadastros saber qual originou o envio |
| `X-Signature` | HMAC-SHA256 do corpo exato da requisição, em hexadecimal, prefixado por `sha256=` |
| `X-Timestamp` | Instante do envio, em ISO 8601. Permite ao cliente detectar replay attack |

**Corpo**

```json
{
  "event_id": "a7f3c811-24d9-4b60-8e15-9c02f7ab5d38",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-05T15:06:59.120Z",
  "order_id": "5c8e0b74-1a92-4d3f-b6e7-08a2c4f19d53",
  "order_number": "ORD-000482",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "9b2f1c34-5a7e-4f21-8d90-3c6b1a2e7f45",
  "total_cents": 148900
}
```

Os itens do pedido **não** são enviados, para manter o payload enxuto. O cliente que precisar
do detalhe consulta `GET /api/v1/orders/:id`.

**Resposta esperada do cliente**

| Faixa | Interpretação da plataforma |
| --- | --- |
| `2xx` | Entrega bem-sucedida. A linha é encerrada |
| Qualquer outra, timeout ou erro de rede | Falha. Entra no fluxo de retry |

**Limites:** corpo de no máximo 64KB; timeout de 10 segundos por tentativa; no máximo 5
tentativas por evento.

---

## 6. Matriz de erros

Todos os códigos usam o prefixo `WEBHOOK_`, seguindo a convenção de
`src/shared/errors/http-errors.ts` ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)).
As classes estendem `AppError`, e o middleware em `src/middlewares/error.middleware.ts` as
serializa sem precisar de alteração.

### 6.1 Erros da API

| Código | Condição | HTTP | Mensagem | Tratamento |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | Endpoint informado não existe | 404 | `Webhook endpoint not found` | Rejeita a requisição |
| `WEBHOOK_INVALID_URL` | URL ausente, malformada ou com esquema diferente de `https` | 400 | `Webhook URL must be a valid https URL` | Rejeita no schema Zod, antes do service |
| `WEBHOOK_INVALID_STATUS_FILTER` | Lista de eventos contém valor fora de `OrderStatus`, ou está vazia | 400 | `Event filter contains an invalid order status` | Rejeita no schema Zod |
| `WEBHOOK_SECRET_REQUIRED` | Operação que exige secret vigente em endpoint sem secret ativa | 422 | `Webhook endpoint has no active secret` | Rejeita e sinaliza estado inconsistente em log |
| `WEBHOOK_ROTATION_IN_PROGRESS` | Nova rotação solicitada dentro da janela de 24h da anterior | 409 | `A secret rotation is already in progress` | Rejeita, informando `previousSecretExpiresAt` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | Replay de identificador inexistente na DLQ | 404 | `Dead letter record not found` | Rejeita a requisição |
| `WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED` | Replay de registro que já possui `replayedAt` | 409 | `Dead letter record has already been replayed` | Rejeita, para não duplicar o evento na outbox |

### 6.2 Erros do fluxo de entrega

Não viram resposta HTTP porque ocorrem dentro do worker. São persistidos em
`webhook_deliveries.error` e em `webhook_outbox.lastError`, e aparecem no histórico de
entregas.

| Código | Condição | Tratamento |
| --- | --- | --- |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | Payload serializado ultrapassa 64KB na montagem | Lançado **dentro da transação** de `changeStatus`, provocando rollback. É o único erro deste bloco que também vira resposta HTTP, com status 422 |
| `WEBHOOK_DELIVERY_TIMEOUT` | Endpoint do cliente não respondeu em 10 segundos | Conta como tentativa falha e agenda retry |
| `WEBHOOK_DELIVERY_FAILED` | Resposta com status fora da faixa 2xx | Conta como tentativa falha e agenda retry. Persiste o status recebido |
| `WEBHOOK_DELIVERY_UNREACHABLE` | Erro de rede, DNS ou TLS antes de obter resposta | Conta como tentativa falha e agenda retry |
| `WEBHOOK_MAX_ATTEMPTS_EXCEEDED` | Quinta tentativa falhou | Move o evento para `webhook_dead_letter` e marca a linha da outbox como `FAILED` |

---

## 7. Estratégias de resiliência

**Timeouts**

| Operação | Valor |
| --- | --- |
| Requisição HTTP ao endpoint do cliente | 10 segundos, via `AbortSignal.timeout` |
| Ciclo de polling do worker | 2 segundos entre ciclos |

**Retry e backoff**

- Máximo de 5 tentativas por evento.
- Progressão fixa: `1min`, `5min`, `30min`, `2h`, `12h`. Tabela, não fórmula.
- Janela total de aproximadamente 15 horas entre a primeira falha e a última tentativa.
- Após a quinta falha, o evento vai para a dead letter queue e não é mais retentado
  automaticamente.

**Degradação e fallback**

- Não há fallback de canal. Se a entrega falha em definitivo, o evento fica na DLQ aguardando
  replay manual. A alternativa de notificação por email está adiada para fase futura.
- Endpoint desativado ou removido durante a vida do evento: a linha é encerrada como `SKIPPED`
  em vez de gerar tentativa.
- Worker fora do ar: os eventos se acumulam como `PENDING` e são processados quando ele
  retorna. Nada é perdido, mas nada é entregue, e nenhuma requisição HTTP falha por isso. É
  o cenário que a métrica de idade do evento pendente mais antigo existe para detectar.

**Invariantes**

1. Se `changeStatus` commitou e existia endpoint ativo interessado no status de destino, existe
   linha correspondente em `webhook_outbox`.
2. Se a transação sofreu rollback, não existe linha de evento correspondente.
3. Todas as tentativas de um mesmo evento usam o mesmo `X-Event-Id`, inclusive após replay.
4. Um evento nunca some: ele está em `PENDING`, `PROCESSING`, `DELIVERED` ou em
   `webhook_dead_letter`.
5. A secret nunca aparece em log, nem em resposta de leitura.

---

## 8. Observabilidade

> O projeto hoje tem apenas log estruturado com Pino. Não há biblioteca de métricas nem de
> tracing nas dependências de `package.json`. As métricas e o tracing abaixo são **propostas
> desta feature (hipótese)**, e a escolha da biblioteca precisa ser fechada na implementação.
> Os logs abaixo usam o que já existe.

**Métricas**

| Métrica | Tipo | Labels | Valor saudável |
| --- | --- | --- | --- |
| `webhook_outbox_pending_age_seconds` | gauge | — | Abaixo de 10. É o indicador mais importante: mede o atraso real percebido pelo cliente |
| `webhook_outbox_pending_total` | gauge | — | Próximo de zero em regime normal |
| `webhook_delivery_attempts_total` | counter | `result`, `response_status` | Proporção de `SUCCESS` acima de 99% |
| `webhook_delivery_duration_seconds` | histogram | `endpoint_id` | p95 abaixo de 2s |
| `webhook_dead_letter_total` | counter | `endpoint_id` | Zero em regime normal. Qualquer incremento merece atenção |
| `webhook_worker_cycle_duration_seconds` | histogram | — | Bem abaixo dos 2s do intervalo de polling |

**Logs**

- Formato: JSON estruturado, com o Pino já configurado em `src/shared/logger/index.ts`.
- Campos obrigatórios em todo log do módulo: `eventId`, `webhookId`, `orderId`, `attempt`,
  `result`.
- Correlação: o `requestId` gerado por `src/middlewares/request-logger.middleware.ts` é
  propagado para dentro da linha da outbox e reaparece nos logs do worker, ligando a
  requisição que mudou o status à entrega correspondente.
- Níveis: `info` para entrega bem-sucedida e ciclo do worker; `warn` para tentativa falha com
  retry agendado; `error` para entrada em DLQ e para estado inconsistente.
- **Nunca logar:** a secret, o `previousSecret`, o valor de `X-Signature` e o payload completo
  do evento. A lista `redactPaths` em `src/shared/logger/index.ts` cobre hoje `authorization`,
  `cookie`, `password`, `passwordHash`, `token` e `accessToken`, e **precisa ser estendida**
  com os campos deste módulo. Sem isso, o reuso do logger dá falsa sensação de proteção.
- O worker precisa sobrescrever `base.service`, hoje fixo em `order-management-api`, para que
  os logs dos dois processos sejam distinguíveis na ferramenta de observabilidade.

**Tracing** *(hipótese)*

- Span raiz por ciclo do worker, com spans filhos por evento processado.
- Span dedicado à requisição HTTP ao endpoint do cliente, com `http.status_code` e duração.
- Propagação do contexto de trace da requisição que originou a mudança de status até a entrega,
  usando o `requestId` como correlação mínima enquanto não houver tracing distribuído de fato.
- Amostragem: 100% para eventos que falharam, taxa reduzida para o caminho feliz.

**Dashboards e alertas**

| Alerta | Gatilho |
| --- | --- |
| Worker parado | `webhook_outbox_pending_age_seconds` acima de 60 por 2 minutos |
| Cliente com problema | 3 falhas consecutivas para o mesmo `endpoint_id` |
| Entrada em DLQ | Qualquer incremento de `webhook_dead_letter_total` |

---

## 9. Integração com o sistema existente

### 9.1 `src/modules/orders/order.service.ts`

**O que faz hoje:** o método `changeStatus` abre uma transação Prisma (`this.prisma.$transaction`)
que carrega o pedido, valida a transição, movimenta estoque quando aplicável, atualiza `orders`
e insere em `order_status_history`.

**Como a feature se integra:** acrescenta uma chamada a
`publishWebhookEvent(tx, order, from, to)` dentro do mesmo `tx`, logo após a inserção em
`order_status_history` e antes do `findUnique` final que devolve o pedido atualizado. A função
recebe o `Prisma.TransactionClient` diretamente, em vez de o service passar a depender de um
repository de webhooks injetado no construtor. Essa escolha mantém a assinatura do
`OrderService` intacta: o construtor continua recebendo apenas `OrderRepository` e
`PrismaClient`.

### 9.2 `src/modules/orders/order.status.ts`

**O que faz hoje:** define o mapa de transições válidas (`transitions`) e expõe `canTransition`,
`allowedTransitions`, `isTerminal`, `shouldDebitStock` e `shouldReplenishStock`.

**Como a feature se integra:** por leitura, não por alteração. Os valores válidos do filtro de
eventos de um webhook são exatamente os do enum `OrderStatus`, e o schema Zod do módulo deriva
dele. O arquivo permanece intocado; o módulo de webhooks apenas consome a mesma fonte de
verdade das transições, o que garante que um filtro não possa referenciar um status que a
máquina de estados desconhece.

### 9.3 `src/shared/errors/http-errors.ts` e `src/shared/errors/app-error.ts`

**O que fazem hoje:** `AppError` carrega `statusCode`, `errorCode` e `details`. Sobre ela vivem
os erros HTTP genéricos (`NotFoundError`, `ConflictError`, `ValidationError`,
`UnprocessableEntityError`) e os de domínio (`InvalidStatusTransitionError`,
`InsufficientStockError`), que seguem a convenção de código em `SCREAMING_SNAKE_CASE`.

**Como a feature se integra:** o módulo cria suas classes de erro seguindo o mesmo padrão,
estendendo a classe HTTP mais próxima. `WebhookNotFoundError` estende `NotFoundError`;
`WebhookRotationInProgressError` estende `ConflictError` com o código
`WEBHOOK_ROTATION_IN_PROGRESS`; `WebhookPayloadTooLargeError` estende
`UnprocessableEntityError`. Nenhuma alteração nos arquivos existentes.

### 9.4 `src/middlewares/error.middleware.ts`

**O que faz hoje:** converte `AppError` em resposta `{ error: { code, message, details } }` com
o `statusCode` da própria classe, e trata `ZodError` e erros conhecidos do Prisma.

**Como a feature se integra:** **sem nenhuma alteração**. Como os erros novos estendem
`AppError`, o primeiro `if` do middleware já os serializa corretamente. Essa é a prova prática
de que a decisão de reuso ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)) tem retorno
concreto: uma família inteira de erros novos sem tocar em infraestrutura compartilhada.

### 9.5 `src/middlewares/auth.middleware.ts`

**O que faz hoje:** `authenticate` valida o JWT e popula `req.user` com `id`, `email` e `role`;
`requireRole(...roles)` rejeita com `ForbiddenError` quem não tem o papel exigido.

**Como a feature se integra:** o router do módulo aplica `authenticate` no topo, como
`src/modules/orders/order.routes.ts` faz. O router administrativo aplica adicionalmente
`requireRole('ADMIN')` na rota de replay. O `req.user.id` é persistido como `replayedBy` no
registro da DLQ, atendendo à exigência de auditoria de quem executou o replay.

### 9.6 `src/middlewares/validate.middleware.ts`

**O que faz hoje:** aplica schemas Zod a `body`, `query` e `params`, convertendo `ZodError` em
`ValidationError` com a lista de campos e mensagens.

**Como a feature se integra:** as rotas do módulo declaram seus schemas em
`src/modules/webhooks/webhook.schemas.ts` **(novo)** e os aplicam pelo `validate` existente. A
exigência de `https` na URL e a validação da lista de eventos contra o enum `OrderStatus` são
regras de schema, não lógica de service.

### 9.7 `src/shared/logger/index.ts`

**O que faz hoje:** cria o logger Pino com `base: { service: 'order-management-api', env }`,
timestamp ISO e a lista `redactPaths` com `req.headers.authorization`, `req.headers.cookie`,
`*.password`, `*.passwordHash`, `*.token` e `*.accessToken`.

**Como a feature se integra:** duas alterações necessárias, ambas identificadas como
consequência negativa em [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md).
Primeiro, `redactPaths` ganha os campos sensíveis do módulo (`*.secret`, `*.previousSecret`,
`*.signature`). Segundo, o worker precisa de um logger com `base.service` próprio, senão os
logs dos dois processos ficam indistinguíveis. A forma mais direta é o worker chamar
`createLogger()`, que já é exportado, com a base sobrescrita.

### 9.8 `src/routes/index.ts`

**O que faz hoje:** `buildApiRouter` recebe o objeto `Controllers` e monta os cinco roteadores
de módulo em `/auth`, `/users`, `/customers`, `/products` e `/orders`.

**Como a feature se integra:** o tipo `Controllers` ganha a entrada `webhooks`, e
`buildApiRouter` passa a montar `buildWebhookRouter` em `/webhooks` e o roteador
administrativo em `/admin/webhooks`. A instanciação da cadeia repository, service e controller
entra em `buildControllers`, em `src/app.ts`, seguindo o mesmo formato dos módulos atuais.

### 9.9 `prisma/schema.prisma`

**O que faz hoje:** define os enums `UserRole` e `OrderStatus` e os modelos `User`, `Customer`,
`Product`, `Order`, `OrderItem`, `OrderStatusHistory` e `OrderNumberSequence`. Todos os
identificadores são `String @id @default(uuid()) @db.Char(36)`, e todos os modelos usam `@@map`
para o nome de tabela em snake case.

**Como a feature se integra:** acrescenta um enum `WebhookEventStatus`
(`PENDING`, `PROCESSING`, `DELIVERED`, `FAILED`) e quatro modelos, seguindo as mesmas
convenções de identificador, `@@map` e índices explícitos:

| Modelo | Tabela | Índices relevantes |
| --- | --- | --- |
| `WebhookEndpoint` | `webhook_endpoints` | `customerId`, `active` |
| `WebhookOutboxEvent` | `webhook_outbox` | `status`, `nextRetryAt`, `createdAt` |
| `WebhookDelivery` | `webhook_deliveries` | `outboxEventId`, `webhookEndpointId`, `attemptedAt` |
| `WebhookDeadLetter` | `webhook_dead_letter` | `webhookEndpointId`, `failedAt` |

O índice composto sobre `status` e `nextRetryAt` em `webhook_outbox` é o que sustenta a
consulta do worker a cada 2 segundos. `WebhookEndpoint` referencia `Customer`, o que exige
acrescentar a relação inversa no modelo `Customer` existente.

### 9.10 `src/app.ts` e `src/server.ts`

**O que fazem hoje:** `src/app.ts` monta o Express, aplica `express.json({ limit: '1mb' })`,
o `requestLogger`, o `/health`, o router em `/api/v1` e o `errorMiddleware`. `src/server.ts` é
a entry-point que sobe o servidor.

**Como a feature se integra:** `src/app.ts` ganha a instanciação do módulo em
`buildControllers`. `src/server.ts` permanece intocado, e ao lado dele nasce `src/worker.ts`
**(novo)**, com um script `npm run worker` em `package.json`. O limite de `1mb` do
`express.json` continua valendo para as requisições de entrada e é independente do limite de
64KB do payload de saída.

---

## 10. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
| --- | --- | --- |
| Node.js | 20 | Já exigido em `package.json`. Dá `fetch` nativo, `AbortSignal.timeout` e `node:crypto` |
| MySQL | a versão em uso | Nenhum recurso novo do banco é necessário |
| Prisma / `@prisma/client` | 5.22.0 | Já presente. Migration nova para as quatro tabelas |
| `zod` | 3.23.8 | Já presente. Schemas do módulo |
| `pino` | 9.5.0 | Já presente. Precisa da extensão de `redactPaths` |
| `uuid` | 11.0.3 | Já presente. Geração do `X-Event-Id` |
| Biblioteca de métricas | a definir | **Não existe no projeto.** Necessária para a seção de observabilidade |
| Biblioteca de tracing | a definir | **Não existe no projeto.** Proposta desta feature |

**Nenhuma dependência nova de runtime é necessária** para o núcleo da feature: HMAC vem do
`node:crypto`, a chamada HTTP vem do `fetch` nativo. As duas linhas em aberto na tabela são de
observabilidade, não de funcionamento.

**Garantias de compatibilidade**

- Nenhum contrato existente muda. Os endpoints atuais respondem exatamente como respondem hoje.
- A assinatura pública do `OrderService` não muda.
- A migration é puramente aditiva: cria tabelas e uma relação inversa, sem alterar coluna
  existente.
- A API mantém o versionamento `/api/v1`; os endpoints novos entram sob o mesmo prefixo.
- O sistema funciona com o worker parado: pedidos mudam de status normalmente e os eventos se
  acumulam para entrega posterior.

---

## 11. Critérios de aceite técnicos

**Consistência transacional**

- Falha injetada na inserção da outbox provoca rollback completo: o status do pedido, o
  histórico e o estoque permanecem inalterados.
- Commit bem-sucedido com endpoint ativo interessado no status resulta em exatamente uma linha
  em `webhook_outbox` por endpoint correspondente.
- Mudança de status sem nenhum endpoint interessado não cria nenhuma linha.

**Entrega**

- Evento pendente é entregue em no máximo 10 segundos após o commit, no caminho feliz.
- Todas as tentativas do mesmo evento enviam o mesmo valor em `X-Event-Id`.
- O corpo enviado é idêntico ao persistido na criação, mesmo quando o pedido muda depois.
- O `X-Signature` enviado confere com o HMAC-SHA256 do corpo calculado com a secret vigente do
  endpoint.

**Retry e DLQ**

- Endpoint respondendo `503` é retentado exatamente 5 vezes, com os intervalos de 1min, 5min,
  30min, 2h e 12h.
- Após a quinta falha existe uma linha em `webhook_dead_letter` com o payload e o motivo da
  última falha, e a linha da outbox está em `FAILED`.
- Endpoint que responde após timeout de 10 segundos é tratado como falha, e a tentativa é
  registrada com a duração real.
- Replay de um registro da DLQ recria a linha na outbox com `attempts = 0` e o mesmo
  `X-Event-Id`, e marca `replayedAt` e `replayedBy`.

**Segurança**

- URL com esquema `http` é rejeitada com `WEBHOOK_INVALID_URL`, no schema, antes do service.
- A secret aparece apenas na resposta de criação e na de rotação. Nenhuma leitura a retorna.
- Nenhum log de nenhum dos dois processos contém secret, `previousSecret` ou assinatura.
- Rotação mantém a secret anterior registrada por exatamente 24 horas.
- Replay sem role `ADMIN` retorna `403`, e com `ADMIN` registra o identificador do executor.

**Padrões do projeto**

- Todos os códigos de erro do módulo começam com `WEBHOOK_`.
- `src/middlewares/error.middleware.ts` não foi alterado, e ainda assim serializa todos os
  erros novos no formato padrão.
- Payload que ultrapassa 64KB dispara `WEBHOOK_PAYLOAD_TOO_LARGE` e provoca rollback.

---

## 12. Riscos e mitigação

### Worker fora do ar sem ninguém perceber

- **Probabilidade:** média
- **Impacto:** alto. Nenhum cliente é notificado, e nenhuma requisição HTTP falha, então o
  sintoma não aparece nos indicadores existentes da API.
- **Mitigação:**
  - `webhook_outbox_pending_age_seconds` com alerta acima de 60 segundos
  - health check próprio do processo do worker
  - log de heartbeat por ciclo, permitindo detectar ausência de atividade
- **Plano de contingência:** reiniciar o worker. Nada é perdido, porque o estado vive no banco;
  a fila drena sozinha.

### Defeito no módulo de webhooks bloqueia mudança de status

- **Probabilidade:** baixa
- **Impacto:** alto. A operação central do domínio fica indisponível.
- **Mitigação:**
  - `publishWebhookEvent` mantida mínima: consulta os endpoints, monta o payload e insere.
    Nenhuma chamada externa, nenhuma regra complexa
  - teste de integração cobrindo o caminho transacional com falha injetada
  - revisão específica do trecho acrescentado em `changeStatus`
- **Plano de contingência:** desativar a inserção por flag de ambiente, aceitando perder
  eventos no período, para restaurar a operação de pedidos.

### Secret exposta em log

- **Probabilidade:** média
- **Impacto:** alto. Um terceiro passa a forjar notificações válidas para aquele endpoint.
- **Mitigação:**
  - extensão de `redactPaths` em `src/shared/logger/index.ts` antes do primeiro deploy
  - proibição explícita de logar o objeto do endpoint inteiro
  - rotação com grace period de 24h, que torna a troca de credencial um procedimento sem
    indisponibilidade
  - revisão de segurança dedicada, com dois dias úteis reservados
- **Plano de contingência:** rotacionar a secret do endpoint afetado e comunicar o cliente.

### Crescimento indefinido da outbox

- **Probabilidade:** alta
- **Impacto:** médio. Degradação progressiva da consulta do worker e do espaço em disco.
- **Mitigação:**
  - índice composto sobre `status` e `nextRetryAt`, que mantém a consulta barata mesmo com a
    tabela grande
  - métrica de contagem total de linhas acompanhada desde o lançamento
- **Plano de contingência:** implementar o arquivamento antes do previsto. A janela ainda é
  questão em aberto no [RFC](RFC.md).

### Cliente não implementa deduplicação

- **Probabilidade:** média
- **Impacto:** médio. Efeito colateral duplicado no sistema do cliente.
- **Mitigação:**
  - `X-Event-Id` estável entre tentativas e preservado no replay
  - documentação destacada no portal do desenvolvedor
- **Plano de contingência:** nenhum do lado da plataforma. É uma consequência assumida do
  modelo at-least-once ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).
