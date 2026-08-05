# RFC: Sistema de Webhooks de Notificação de Pedidos

| | |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | 2026-08-05 |
| **Revisores** | Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança), Marcos (Product Manager) |

---

## TL;DR

Três clientes B2B pediram para ser notificados quando o status dos pedidos deles muda, em vez
de continuar fazendo polling em `GET /orders`. Propomos entregar isso com o padrão **Outbox**:
o evento é gravado em uma tabela do MySQL dentro da mesma transação que altera o status, e um
**worker em processo separado** consome essa tabela por polling de 2 segundos e faz a chamada
HTTP. Falhas são retentadas 5 vezes com backoff exponencial e, esgotadas as tentativas, o
evento vai para uma dead letter queue com replay manual. Cada requisição é assinada com
**HMAC-SHA256** usando uma secret exclusiva do endpoint.

O custo desta proposta: a entrega é **at-least-once**, o que transfere a deduplicação para o
cliente; a latência mínima é de 2 segundos; e ganhamos um processo novo para operar e
monitorar. Estimativa de três sprints, incluindo a revisão de segurança.

---

## Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) formalizaram o pedido de
notificação em tempo real sobre mudança de status dos pedidos. Hoje eles consultam
`GET /orders` periodicamente, o que torna a integração lenta e cara do lado deles. A Atlas
sinalizou que pode migrar para um concorrente se a entrega não sair até o fim do trimestre.

Questionados sobre o que entendem por tempo real, os clientes responderam que qualquer
latência abaixo de **10 segundos** atende. O que não pode acontecer é a notificação ficar
pendurada e obrigá-los a atualizar manualmente.

O escopo é **outbound apenas**: a plataforma notifica os clientes, e não recebe webhooks
deles.

A dificuldade técnica está em onde o evento nasce. A mudança de status acontece no método
`changeStatus` de `src/modules/orders/order.service.ts`, dentro de uma transação Prisma que já
faz três operações: atualiza a `orders`, insere em `order_status_history` e movimenta o
estoque dos produtos quando a transição exige.

Introduzir uma chamada HTTP nessa transação criaria dois problemas sem saída boa. Se o
endpoint do cliente estiver lento, ele segura a transação e trava a mudança de status de
outros pedidos. Se estiver fora do ar, a única alternativa seria dar rollback em uma mudança
de status legítima. Ao mesmo tempo, deixar a notificação inteiramente fora da transação abre a
janela oposta: o status muda, o processo cai, e o evento nunca chega a existir.

A plataforma não tem hoje nenhum mecanismo de evento, fila ou notificação externa, não opera
componente de mensageria, e o time é pequeno.

---

## Proposta técnica

A proposta separa a **garantia de registro** da **garantia de entrega**, resolvendo cada uma
onde ela é barata.

**Registro.** Adotamos o padrão Outbox ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)): quando o
status muda, uma linha de evento é inserida em `webhook_outbox` dentro da mesma transação que
já atualiza `orders` e `order_status_history`. Se a transação commita, o evento existe; se
sofre rollback, o evento some junto. Não há janela de inconsistência possível.

A linha guarda o **payload já renderizado**
([ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)), como snapshot do pedido no
instante da transição. Isso importa porque o envio pode acontecer horas depois, e um evento
que descreve uma transição antiga com dados atuais é internamente incoerente.

**Entrega.** Um worker em processo separado consome a outbox por polling de 2 segundos
([ADR-002](adrs/ADR-002-worker-separado-em-polling.md)). Mesmo banco e mesma stack, mas fora
do processo da API, para que um deploy ou restart da API não derrube a entrega. Enquanto o
worker for único, a entrega chega ao cliente na ordem de criação dos eventos.

**Falha.** Uma tentativa falha quando a resposta não é de sucesso ou quando estoura o timeout.
O evento é retentado 5 vezes com backoff exponencial crescente, cobrindo cerca de 15 horas, o
que absorve manutenções planejadas de cliente. Esgotadas as tentativas, o evento vai para
`webhook_dead_letter`, com payload e motivo da falha, e pode ser reprocessado por um endpoint
administrativo ([ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)).

**Garantia de entrega.** Outbox com retry torna a entrega duplicada possível: o worker não
distingue "não entreguei" de "entreguei mas não soube". Assumimos **at-least-once** e enviamos
um identificador estável por evento no header `X-Event-Id`, para o cliente dedupicar do lado
dele ([ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)). É o modelo de Stripe e
GitHub, o que reduz a fricção de integração.

**Confiança.** Cada requisição é assinada com HMAC-SHA256 sobre o corpo, com secret exclusiva
daquele endpoint, rotacionável com grace period de 24 horas
([ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)). Por endpoint e não global,
para que o vazamento de uma credencial não comprometa a base inteira.

**Superfície de API.** O cliente gerencia seus endpoints por CRUD autenticado, escolhendo
quais status quer ouvir, e consulta o histórico de entregas. O filtro de status é aplicado na
**inserção** da outbox: se nenhum webhook do cliente quer aquele status, a linha nem é criada.
O replay de dead letter exige role `ADMIN`.

**Encaixe no código.** O módulo segue a estrutura dos cinco módulos existentes e reaproveita
`AppError`, o error middleware, o `validate` com Zod, o `requireRole` e o logger Pino, com
todos os códigos de erro prefixados por `WEBHOOK_`
([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)). Contratos, matriz de erros e
detalhamento de cada fluxo estão no [FDD](FDD.md).

### Escopo desta proposta

**Incluído**

- Gravação transacional do evento na outbox a partir da mudança de status
- Worker de entrega com retry, backoff e dead letter queue
- Assinatura HMAC-SHA256 com secret por endpoint e rotação
- CRUD de configuração de webhook, com filtro por status
- Consulta de histórico de entregas
- Endpoint administrativo de replay de dead letter

**Não incluído nesta fase**

- **Notificação por email ao cliente** quando o webhook dele falha de forma recorrente.
  Adiado para depois de medirmos o impacto real.
- **Dashboard visual** para o cliente acompanhar os webhooks. É projeto separado do time de
  frontend; nesta fase entregamos apenas endpoints.
- **Arquivamento das linhas entregues** da outbox.
- **Webhooks de entrada**, do cliente para a plataforma.

---

## Alternativas consideradas

### Disparo síncrono da chamada HTTP dentro de `changeStatus`

Emitir a notificação diretamente no service, no momento da transição.

**Trade-off que levou ao descarte:** acopla a disponibilidade e a latência de um sistema de
terceiro a uma operação central do domínio. Um cliente lento travaria a mudança de status de
outros pedidos, e um cliente fora do ar deixaria como única saída o rollback de uma transição
legítima.

### Redis Streams ou componente de mensageria dedicado

Publicar o evento em um stream e ter consumidores lendo dali.

**Trade-off que levou ao descarte:** exigiria subir e operar infraestrutura que o time não tem
hoje. Para o volume em questão, o MySQL existente resolve, e o custo operacional de um
componente novo não se justifica com o tamanho atual do time.

### Trigger de banco para notificar o worker

Reagir à inserção na outbox em vez de fazer polling.

**Trade-off que levou ao descarte:** o MySQL não tem mecanismo nativo de notificação para
processo externo, diferente do `LISTEN`/`NOTIFY` do PostgreSQL. Uma trigger apenas executa
SQL. Avisar o worker exigiria improvisar algo como escrita em arquivo ou chamada a um
endpoint, acrescentando um mecanismo frágil para ganhar poucos segundos que o requisito não
pede.

### Entrega exactly-once

Garantir que cada evento chegue exatamente uma vez.

**Trade-off que levou ao descarte:** exigiria coordenação transacional entre os dois lados,
transformando uma integração simples por HTTP em um protocolo com estado compartilhado.
At-least-once com identificador de evento cobre os mesmos casos práticos a uma fração da
complexidade.

---

## Questões em aberto

### Rate limiting de envio para o cliente

Se um cliente tiver 50 pedidos mudando de status em um minuto, a plataforma dispara 50
chamadas contra o endpoint dele. Não sabemos se isso é problema real no volume atual.

- **Bloqueia:** nada nesta fase. Afeta a experiência do cliente em picos.
- **Como resolver:** observar em produção depois do lançamento e implementar se virar
  problema.

### Gatilho para a fase de notificação por email

O alerta por email ao cliente com webhook falhando entra em fase futura, "depois que a gente
medir o impacto". O que será medido, e a partir de que número a fase seguinte se justifica,
não foi definido.

- **Bloqueia:** o planejamento da fase 2.
- **Como resolver:** definir a métrica com o Marcos assim que houver dado de produção sobre
  volume de entradas na dead letter queue.

### Estratégia de escala para múltiplos workers

A ordenação por `order_id` existe hoje porque o worker é único. Se a escala exigir
paralelismo, essa garantia se perde. Duas saídas foram levantadas, particionar por `order_id`
ou usar lock pessimista, sem decisão entre elas.

- **Bloqueia:** nada agora. Vira bloqueio quando um worker único não der conta.
- **Como resolver:** decidir com dado de throughput real, antes de a fila começar a acumular.
  A escolha muda a modelagem.

### Política de retenção da outbox

O arquivamento das linhas entregues foi mencionado como "depois de 30 dias ou assim" e
colocado fora do escopo. Sem a janela definida, a tabela cresce indefinidamente.

- **Bloqueia:** nada no lançamento. Vira problema operacional com o acúmulo.
- **Como resolver:** definir a janela junto com a estimativa de volume mensal de eventos,
  antes do primeiro trimestre em produção.

### Autorização do CRUD de configuração

Nesta fase, qualquer role autenticada pode criar, editar e remover configurações de webhook.
Ficou registrado que "mais pra frente a gente pode endurecer", sem definição de quando nem
para qual modelo.

- **Bloqueia:** nada funcionalmente. É dívida de segurança assumida.
- **Como resolver:** revisar com a Sofia depois do lançamento.

---

## Impacto e riscos

### Impacto

- **Transação de mudança de status.** `changeStatus` ganha mais uma escrita no caminho
  crítico. Pequena em custo, mas passa a permitir que um defeito no módulo de webhooks
  bloqueie uma operação de negócio.
- **Operação.** Um processo novo para implantar e manter no ar. Se o worker cair, os eventos
  se acumulam sem que nenhuma requisição HTTP falhe, o que torna o monitoramento do worker
  obrigatório e não opcional.
- **Consumidores atuais da API.** Nenhum. A feature acrescenta endpoints e não altera
  contratos existentes.
- **Clientes integrados.** Precisam implementar verificação de assinatura e deduplicação por
  `X-Event-Id`, o que vira material do portal do desenvolvedor.
- **Time.** Três sprints, incluindo dois dias úteis de revisão de segurança antes do deploy.

### Riscos

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| Worker cai e a falha passa despercebida, acumulando eventos em silêncio | Média | Alto: clientes deixam de ser notificados sem nenhum sinal de erro | Métrica de idade do evento pendente mais antigo, com alerta; health check do processo |
| Cliente não implementa deduplicação e processa eventos repetidos | Média | Médio: efeito colateral no sistema do cliente, com risco de dado duplicado | Documentação destacada no portal; `X-Event-Id` estável entre retentativas |
| Secret vaza em log, do lado da plataforma ou do cliente | Média | Alto: terceiro passa a forjar notificações válidas para aquele endpoint | Secret por endpoint limita o raio; rotação com grace period de 24h; extensão da lista de redação do logger; revisão de segurança antes do deploy |
| Defeito na inserção da outbox bloqueia mudança de status | Baixa | Alto: operação central do domínio indisponível | Lógica de inserção mínima, sem chamada externa; cobertura de teste no caminho transacional |
| Prazo não cabe nas três sprints estimadas | Média | Alto: risco comercial declarado de perda da Atlas | Escopo já enxuto, com quatro itens explicitamente adiados; revisão de segurança agendada com antecedência |

---

## Decisões relacionadas

Cada decisão desta proposta está registrada em um ADR próprio, com contexto, alternativas e
consequências. Índice completo em [`docs/adrs/`](adrs/README.md).

- [ADR-001](adrs/ADR-001-outbox-no-mysql.md) Outbox no MySQL
- [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) Worker separado em polling
- [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md) Retry com backoff e DLQ
- [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) HMAC-SHA256 com secret por endpoint
- [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) At-least-once com X-Event-Id
- [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) Reuso dos padrões existentes
- [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) Snapshot do payload na inserção

---

## Referências

Origem desta proposta: [`TRANSCRICAO.md`](../TRANSCRICAO.md). Documentos irmãos:
[PRD](PRD.md), [FDD](FDD.md), [TRACKER](TRACKER.md).
