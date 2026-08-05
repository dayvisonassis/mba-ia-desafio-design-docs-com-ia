# Tracker de Rastreabilidade

Referência cruzada entre cada item registrado nos documentos deste pacote e sua origem. A
regra que este documento existe para verificar é simples: **todo item registrado tem origem
identificável**. Um item cuja coluna Localização não pode ser preenchida foi inventado, e sai
do documento ou tem a origem corrigida.

**Como ler a coluna Localização**

- Origem `TRANSCRICAO`: timestamp e nome do participante em [`TRANSCRICAO.md`](../TRANSCRICAO.md),
  no formato `[hh:mm] Nome`.
- Origem `CODIGO`: caminho do arquivo no repositório, a partir da raiz.
- Origem `HIPOTESE`: o item **não tem origem na transcrição nem no código**. É uma suposição
  deliberada, marcada como tal também no documento em que aparece.

## Resumo da auditoria

| Verificação | Resultado |
| --- | --- |
| Itens rastreados | 140 |
| Origem `TRANSCRICAO` | 112 (80,0%) |
| Origem `CODIGO` | 23 (16,4%) |
| Origem `HIPOTESE` | 5 (3,6%) |
| Timestamps validados contra a transcrição | 112 de 112 |
| Caminhos de código validados no disco | 23 de 23 |
| Identificadores duplicados | nenhum |

**As 5 hipóteses** são os únicos itens do pacote sem origem em fonte. Estão todos marcados no
texto dos documentos e listados aqui com a justificativa na coluna Localização: a meta de
redução de polling e a meta de disponibilidade, que a reunião não quantificou; as métricas e o
tracing, que o projeto não possui hoje e a reunião não discutiu; dois códigos de erro
derivados de regras que a reunião definiu sem nomear o erro; e a modelagem de uma linha por
par evento e endereço, derivada da combinação de dois pontos da reunião.

## Matriz

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | `docs/PRD.md` | Restrição | Pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | `docs/PRD.md` | Restrição | Clientes fazem polling em GET /orders, deixando a integração lenta e cara do lado deles | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-03 | `docs/PRD.md` | Restrição | Risco comercial: Atlas pode migrar para concorrente se não entregar até o fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Revisão de segurança com no mínimo dois dias úteis antes do deploy, bloqueante | TRANSCRICAO | `[09:46] Sofia` |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Documentação do comportamento de entrega repetida no portal do desenvolvedor | TRANSCRICAO | `[09:26] Marcos` |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Implementação do lado do cliente: verificar assinatura e deduplicar por identificador | TRANSCRICAO | `[09:25] Diego` |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | Definição das ferramentas de métricas e tracing, inexistentes no projeto | CODIGO | `package.json` |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastro de endereço de notificação com url, filtro de status e credencial devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Consulta dos endereços cadastrados de um cliente | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Edição de url, filtro de status e estado ativo de um endereço | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Remoção de endereço de notificação | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Filtro por status aplicado na geração da notificação, não no envio | TRANSCRICAO | `[09:34] Bruno` |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Notificação automática gerada na mesma operação que altera o status do pedido | TRANSCRICAO | `[09:40] Bruno` |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Autenticação da notificação por assinatura, permitindo ao cliente comprovar origem e integridade | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Rotação da credencial com a anterior válida por 24 horas em paralelo | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Repetição automática em caso de falha, cinco tentativas com espaçamento crescente | TRANSCRICAO | `[09:15] Diego` |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Registro em separado das notificações que esgotaram as tentativas, com motivo da falha | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Consulta do histórico de entregas com sucesso/falha, conteúdo, resposta e tempo | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Reprocessamento manual de falha definitiva, restrito a papel administrativo | TRANSCRICAO | `[09:35] Diego` |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Latência abaixo de 10 segundos, com piso de 2 segundos pelo intervalo de verificação | TRANSCRICAO | `[09:10] Larissa` |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Tempo máximo de espera por resposta do destinatário de 10 segundos | TRANSCRICAO | `[09:42] Diego` |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Disponibilidade mensal de 99,9% do serviço de notificação | HIPOTESE | `meta não definida na reunião` |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Assinatura HMAC-SHA256 obrigatória em toda notificação enviada | TRANSCRICAO | `[09:20] Sofia` |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | Destino precisa usar canal seguro; endereço sem TLS é recusado na validação | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | Conteúdo da notificação limitado a 64KB, com erro em vez de truncamento | TRANSCRICAO | `[09:24] Larissa` |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | Registro e mudança de status atômicos: ou os dois acontecem, ou nenhum | TRANSCRICAO | `[09:40] Bruno` |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Entrega ao menos uma vez, com identificador único e estável por evento | TRANSCRICAO | `[09:24] Diego` |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Ordem de entrega preservada por pedido enquanto o processamento for sequencial | TRANSCRICAO | `[09:13] Larissa` |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | Reprocessamento exige papel administrativo e registra o autor para auditoria | TRANSCRICAO | `[09:36] Sofia` |
| PRD-NFR-11 | `docs/PRD.md` | Requisito Não Funcional | Nenhum contrato existente da API é alterado | CODIGO | `src/routes/index.ts` |
| PRD-OBJ-01 | `docs/PRD.md` | Objetivo | Latência p95 abaixo de 10 segundos entre mudança de status e primeira tentativa de entrega | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-02 | `docs/PRD.md` | Objetivo | Redução de pelo menos 80% nas chamadas de polling dos clientes integrados | HIPOTESE | `meta não definida na reunião; problema em [09:00] Marcos` |
| PRD-OBJ-03 | `docs/PRD.md` | Objetivo | Zero eventos gerados sem entrega confirmada e sem registro de falha | TRANSCRICAO | `[09:06] Diego` |
| PRD-OBJ-04 | `docs/PRD.md` | Objetivo | Janela de aproximadamente 15 horas coberta pelas tentativas automáticas | TRANSCRICAO | `[09:17] Diego` |
| PRD-OBJ-05 | `docs/PRD.md` | Objetivo | Três de três clientes integrados até o fim de novembro | TRANSCRICAO | `[09:45] Marcos` |
| PRD-OBJ-06 | `docs/PRD.md` | Objetivo | Entrega em três sprints, incluindo a revisão de segurança | TRANSCRICAO | `[09:46] Larissa` |
| PRD-OUT-01 | `docs/PRD.md` | Exclusão | Notificação por email ao cliente com webhook falhando, adiada para fase futura | TRANSCRICAO | `[09:37] Larissa` |
| PRD-OUT-02 | `docs/PRD.md` | Exclusão | Dashboard visual descartado nesta fase, projeto separado do time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| PRD-OUT-03 | `docs/PRD.md` | Exclusão | Rate limiting das notificações deixado em aberto para observação em produção | TRANSCRICAO | `[09:39] Diego` |
| PRD-OUT-04 | `docs/PRD.md` | Exclusão | Arquivamento das notificações entregues fora do escopo desta feature | TRANSCRICAO | `[09:08] Diego` |
| PRD-OUT-05 | `docs/PRD.md` | Exclusão | Webhooks de entrada fora de escopo: os clientes querem receber, não enviar | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OUT-06 | `docs/PRD.md` | Exclusão | Processamento paralelo adiado, porque a ordem de entrega depende de ser sequencial | TRANSCRICAO | `[09:13] Diego` |
| PRD-RISCO-01 | `docs/PRD.md` | Risco | Processo de envio para de funcionar sem nenhum indicador da API acusar problema | TRANSCRICAO | `[09:11] Diego` |
| PRD-RISCO-02 | `docs/PRD.md` | Risco | Defeito no módulo de notificação bloqueia a mudança de status por rollback | TRANSCRICAO | `[09:40] Bruno` |
| PRD-RISCO-03 | `docs/PRD.md` | Risco | Vazamento de credencial de assinatura, com caso real anterior de cliente | TRANSCRICAO | `[09:22] Diego` |
| PRD-RISCO-04 | `docs/PRD.md` | Risco | Cliente não implementa deduplicação e processa notificações repetidas | TRANSCRICAO | `[09:25] Sofia` |
| PRD-RISCO-05 | `docs/PRD.md` | Risco | Prazo de três sprints não se sustenta, com risco comercial declarado | TRANSCRICAO | `[09:45] Marcos` |
| PRD-TEST-01 | `docs/PRD.md` | Estratégia de teste | Testes de integração seguindo o padrão atual com Vitest e Supertest sobre banco de teste | CODIGO | `vitest.config.ts` |
| PRD-TEST-02 | `docs/PRD.md` | Estratégia de teste | Testes ponta a ponta da integração no order.service previstos na estimativa | TRANSCRICAO | `[09:46] Larissa` |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa considerada | Disparo síncrono dentro de changeStatus, descartado por acoplar disponibilidade de terceiro a operação central | TRANSCRICAO | `[09:04] Bruno` |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa considerada | Redis Streams ou mensageria dedicada, descartado por exigir infraestrutura que o time não opera | TRANSCRICAO | `[09:07] Diego` |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa considerada | Trigger de banco para notificar o worker, descartada por limitação do MySQL | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-04 | `docs/RFC.md` | Alternativa considerada | Entrega exactly-once, descartada por exigir coordenação transacional entre os dois lados | TRANSCRICAO | `[09:25] Diego` |
| RFC-CTX-01 | `docs/RFC.md` | Restrição | Pedido formal de três clientes B2B: Atlas Comercial, MaxDistribuição e Nova Cargo | TRANSCRICAO | `[09:00] Marcos` |
| RFC-CTX-02 | `docs/RFC.md` | Restrição | Risco comercial declarado: Atlas pode migrar para concorrente se não entregar até fim do trimestre | TRANSCRICAO | `[09:00] Marcos` |
| RFC-CTX-03 | `docs/RFC.md` | Requisito Não Funcional | Latência abaixo de 10 segundos é considerada tempo real pelos clientes | TRANSCRICAO | `[09:02] Marcos` |
| RFC-CTX-04 | `docs/RFC.md` | Restrição | Escopo outbound apenas: a plataforma notifica, não recebe webhooks dos clientes | TRANSCRICAO | `[09:02] Marcos` |
| RFC-CTX-05 | `docs/RFC.md` | Restrição | Transação de changeStatus já atualiza orders, insere no history e movimenta estoque | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-IMP-01 | `docs/RFC.md` | Impacto | Estimativa de três sprints incluindo dois dias úteis de revisão de segurança | TRANSCRICAO | `[09:46] Larissa` |
| RFC-META-01 | `docs/RFC.md` | Metadado | Larissa como autora e Bruno, Diego, Sofia e Marcos como revisores, nomeados na própria reunião | TRANSCRICAO | `[09:50] Larissa` |
| RFC-OPEN-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de envio para o cliente: observar em produção e decidir depois | TRANSCRICAO | `[09:39] Diego` |
| RFC-OPEN-02 | `docs/RFC.md` | Questão em aberto | Gatilho para a fase de notificação por email: qual métrica medir e a partir de que número | TRANSCRICAO | `[09:37] Larissa` |
| RFC-OPEN-03 | `docs/RFC.md` | Questão em aberto | Estratégia de escala para múltiplos workers: particionar por order_id ou lock pessimista | TRANSCRICAO | `[09:13] Diego` |
| RFC-OPEN-04 | `docs/RFC.md` | Questão em aberto | Política de retenção da outbox: janela mencionada como 30 dias ou assim, sem definição | TRANSCRICAO | `[09:08] Diego` |
| RFC-OPEN-05 | `docs/RFC.md` | Questão em aberto | Autorização do CRUD de configuração: aberta a qualquer role autenticada, a endurecer depois | TRANSCRICAO | `[09:37] Sofia` |
| RFC-OUT-01 | `docs/RFC.md` | Exclusão | Notificação por email ao cliente com webhook falhando, adiada para fase futura | TRANSCRICAO | `[09:37] Larissa` |
| RFC-OUT-02 | `docs/RFC.md` | Exclusão | Dashboard visual para o cliente, projeto separado do time de frontend | TRANSCRICAO | `[09:40] Larissa` |
| RFC-OUT-03 | `docs/RFC.md` | Exclusão | Arquivamento das linhas entregues da outbox, fora do escopo desta feature | TRANSCRICAO | `[09:08] Diego` |
| RFC-OUT-04 | `docs/RFC.md` | Exclusão | Webhooks de entrada, do cliente para a plataforma | TRANSCRICAO | `[09:02] Marcos` |
| RFC-RISCO-01 | `docs/RFC.md` | Risco | Worker cai e a falha passa despercebida, acumulando eventos sem nenhum sinal de erro | TRANSCRICAO | `[09:11] Diego` |
| RFC-RISCO-02 | `docs/RFC.md` | Risco | Cliente não implementa deduplicação e processa eventos repetidos | TRANSCRICAO | `[09:25] Sofia` |
| RFC-RISCO-03 | `docs/RFC.md` | Risco | Secret vaza em log e terceiro passa a forjar notificações válidas | TRANSCRICAO | `[09:22] Diego` |
| RFC-RISCO-04 | `docs/RFC.md` | Risco | Defeito na inserção da outbox bloqueia a mudança de status por rollback | TRANSCRICAO | `[09:40] Bruno` |
| RFC-RISCO-05 | `docs/RFC.md` | Risco | Prazo não cabe nas três sprints, com risco comercial declarado de perda da Atlas | TRANSCRICAO | `[09:45] Marcos` |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | POST /api/v1/webhooks cadastra endpoint com url, events e customerId; secret gerada e devolvida na criação | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | GET /api/v1/webhooks lista os endpoints de um customer | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | PATCH /api/v1/webhooks/:id edita url, filtro de eventos e estado ativo | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | DELETE /api/v1/webhooks/:id remove a configuração do endpoint | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | POST /api/v1/webhooks/:id/rotate-secret gera nova secret com grace period de 24h | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | GET /api/v1/webhooks/:id/deliveries retorna histórico com sucesso/falha, payload, response e tempo | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | POST /api/v1/admin/webhooks/dead-letter/:id/replay exige role ADMIN | TRANSCRICAO | `[09:35] Diego` |
| FDD-CONTRATO-08 | `docs/FDD.md` | Contrato | Requisição de saída com payload JSON contendo event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id e total_cents | TRANSCRICAO | `[09:43] Diego` |
| FDD-CONTRATO-09 | `docs/FDD.md` | Contrato | Headers de saída: X-Event-Id, X-Signature, X-Timestamp, Content-Type e X-Webhook-Id | TRANSCRICAO | `[09:44] Diego` |
| FDD-CONTRATO-10 | `docs/FDD.md` | Contrato | X-Webhook-Id acrescentado para o cliente com vários cadastros saber qual originou o envio | TRANSCRICAO | `[09:44] Sofia` |
| FDD-CONTRATO-11 | `docs/FDD.md` | Contrato | Itens do pedido não entram no payload; o cliente consulta GET /orders/:id se precisar do detalhe | TRANSCRICAO | `[09:43] Diego` |
| FDD-DEP-01 | `docs/FDD.md` | Dependência | Node 20+, Prisma 5.22, zod, pino e uuid já presentes; nenhuma dependência nova de runtime | CODIGO | `package.json` |
| FDD-DEP-02 | `docs/FDD.md` | Dependência | Nenhuma biblioteca de métricas ou tracing existe no projeto; a escolha fica para a implementação | CODIGO | `package.json` |
| FDD-DESIGN-01 | `docs/FDD.md` | Decisão | Uma linha de outbox por par (evento, endpoint), para que o retry seja por destinatário | HIPOTESE | `derivado de [09:44] Sofia e [09:15] Diego` |
| FDD-ERRO-01 | `docs/FDD.md` | Erro previsto | WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL e WEBHOOK_SECRET_REQUIRED como códigos do módulo | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERRO-02 | `docs/FDD.md` | Restrição | Prefixo WEBHOOK_ obrigatório em todos os códigos de erro do módulo | TRANSCRICAO | `[09:29] Larissa` |
| FDD-ERRO-03 | `docs/FDD.md` | Erro previsto | WEBHOOK_INVALID_URL disparado quando a URL não é https, validado no schema Zod | TRANSCRICAO | `[09:23] Sofia` |
| FDD-ERRO-04 | `docs/FDD.md` | Erro previsto | WEBHOOK_PAYLOAD_TOO_LARGE quando o payload ultrapassa 64KB; erra em vez de truncar | TRANSCRICAO | `[09:24] Larissa` |
| FDD-ERRO-05 | `docs/FDD.md` | Erro previsto | WEBHOOK_DELIVERY_TIMEOUT quando o cliente não responde em 10 segundos | TRANSCRICAO | `[09:42] Diego` |
| FDD-ERRO-06 | `docs/FDD.md` | Erro previsto | WEBHOOK_MAX_ATTEMPTS_EXCEEDED após a quinta tentativa falha | TRANSCRICAO | `[09:15] Diego` |
| FDD-ERRO-07 | `docs/FDD.md` | Erro previsto | WEBHOOK_DEAD_LETTER_ALREADY_REPLAYED e WEBHOOK_ROTATION_IN_PROGRESS derivados das regras de replay e rotação | HIPOTESE | `derivado de [09:21] Sofia e [09:35] Diego` |
| FDD-FLUXO-01 | `docs/FDD.md` | Fluxo | Inserção do evento na outbox dentro da transação de changeStatus, via publishWebhookEvent(tx, order, from, to) | TRANSCRICAO | `[09:41] Bruno` |
| FDD-FLUXO-02 | `docs/FDD.md` | Fluxo | Filtro de status aplicado na inserção: se nenhum endpoint quer o status, nenhuma linha é criada | TRANSCRICAO | `[09:34] Bruno` |
| FDD-FLUXO-03 | `docs/FDD.md` | Fluxo | Worker seleciona lote de PENDING a cada 2 segundos, ordenado por createdAt | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLUXO-04 | `docs/FDD.md` | Fluxo | Retry incrementa attempts e agenda nextRetryAt pela tabela fixa 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego` |
| FDD-FLUXO-05 | `docs/FDD.md` | Fluxo | Quinta falha move o evento para webhook_dead_letter com payload e motivo da última falha | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLUXO-06 | `docs/FDD.md` | Fluxo | Replay recria a linha na outbox com o mesmo X-Event-Id e registra replayedAt e replayedBy | TRANSCRICAO | `[09:36] Sofia` |
| FDD-FLUXO-07 | `docs/FDD.md` | Fluxo | Rotação move a secret atual para previousSecret com validade de 24 horas | TRANSCRICAO | `[09:21] Sofia` |
| FDD-INT-01 | `docs/FDD.md` | Ponto de integração | changeStatus recebe publishWebhookEvent dentro do mesmo tx, sem alterar a assinatura do OrderService | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Ponto de integração | Enum OrderStatus e o mapa de transições são a fonte de verdade do filtro de eventos; arquivo não é alterado | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | `docs/FDD.md` | Ponto de integração | Erros do módulo estendem NotFoundError, ConflictError e UnprocessableEntityError, seguindo a convenção existente | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-04 | `docs/FDD.md` | Ponto de integração | Error middleware serializa os erros novos sem nenhuma alteração, por herança de AppError | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Ponto de integração | authenticate no router do módulo e requireRole('ADMIN') na rota de replay; req.user.id vira replayedBy | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-06 | `docs/FDD.md` | Ponto de integração | Schemas Zod do módulo aplicados pelo middleware validate existente | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-07 | `docs/FDD.md` | Ponto de integração | redactPaths precisa ganhar secret, previousSecret e signature; worker precisa de base.service próprio | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-08 | `docs/FDD.md` | Ponto de integração | buildApiRouter monta o router do módulo em /webhooks e o administrativo em /admin/webhooks | CODIGO | `src/routes/index.ts` |
| FDD-INT-09 | `docs/FDD.md` | Ponto de integração | Quatro modelos novos seguindo UUID Char(36), @@map em snake case e índices explícitos | CODIGO | `prisma/schema.prisma` |
| FDD-INT-10 | `docs/FDD.md` | Ponto de integração | requestId do request-logger propagado para a outbox e reaparecendo nos logs do worker | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-INT-11 | `docs/FDD.md` | Ponto de integração | Envelope de paginação das listagens segue a função paginated existente | CODIGO | `src/shared/http/response.ts` |
| FDD-INT-12 | `docs/FDD.md` | Ponto de integração | Worker como entry-point nova ao lado de server.ts, acionada por npm run worker | TRANSCRICAO | `[09:11] Larissa` |
| FDD-OBS-01 | `docs/FDD.md` | Requisito Não Funcional | Métricas e tracing propostos por esta feature, sem origem na reunião | HIPOTESE | `proposta do FDD, sem correspondente na transcrição` |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Evento gravado em webhook_outbox na mesma transação que altera o status do pedido | TRANSCRICAO | `[09:06] Diego` |
| ADR-001-ALT-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa considerada | Disparo síncrono da chamada HTTP dentro de changeStatus, descartado por acoplar latência externa à transação | TRANSCRICAO | `[09:04] Bruno` |
| ADR-001-ALT-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa considerada | Redis Streams ou mensageria dedicada, descartado como overengineering para o tamanho do time | TRANSCRICAO | `[09:07] Diego` |
| ADR-001-CTX-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Transação de changeStatus já atualiza orders, insere em order_status_history e movimenta estoque | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-002 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Decisão | Worker em processo separado (src/worker.ts) consumindo a outbox por polling de 2 segundos | TRANSCRICAO | `[09:10] Larissa` |
| ADR-002-ALT-01 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Alternativa considerada | Trigger de banco notificando o worker, descartada porque MySQL não tem listener para processo externo | TRANSCRICAO | `[09:09] Diego` |
| ADR-002-ALT-02 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Alternativa considerada | Worker no mesmo processo da API, descartado porque restart da API derrubaria o worker | TRANSCRICAO | `[09:11] Diego` |
| ADR-002-LIM-01 | `docs/adrs/ADR-002-worker-separado-em-polling.md` | Restrição | Ordenação garantida apenas por order_id e apenas enquanto houver um único worker | TRANSCRICAO | `[09:13] Larissa` |
| ADR-003 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Decisão | 5 tentativas com backoff 1m/5m/30m/2h/12h e DLQ em tabela webhook_dead_letter separada | TRANSCRICAO | `[09:17] Larissa` |
| ADR-003-ALT-01 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa considerada | 3 tentativas, descartado por cobrir apenas 30 minutos e perder eventos em manutenção planejada | TRANSCRICAO | `[09:16] Diego` |
| ADR-003-ALT-02 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa considerada | Retry indefinido com backoff, descartado por deixar evento pendurado se o cliente sumir | TRANSCRICAO | `[09:15] Diego` |
| ADR-003-ALT-03 | `docs/adrs/ADR-003-retry-com-backoff-e-dlq.md` | Alternativa considerada | Marcar failed na própria outbox, descartado por poluir a leitura da tabela principal | TRANSCRICAO | `[09:18] Diego` |
| ADR-004 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Decisão | HMAC-SHA256 sobre o corpo, secret única por endpoint, rotação com grace period de 24h | TRANSCRICAO | `[09:22] Sofia` |
| ADR-004-ALT-01 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Alternativa considerada | Secret global da plataforma, descartada porque um vazamento comprometeria todos os clientes | TRANSCRICAO | `[09:21] Sofia` |
| ADR-004-CTX-01 | `docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md` | Restrição | Logger Pino já redige authorization, token, password e passwordHash, mas não os campos do módulo | CODIGO | `src/shared/logger/index.ts` |
| ADR-005 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Decisão | Garantia at-least-once com X-Event-Id no header e deduplicação de responsabilidade do cliente | TRANSCRICAO | `[09:26] Larissa` |
| ADR-005-ALT-01 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Alternativa considerada | Exactly-once, descartado por exigir coordenação transacional entre os dois lados | TRANSCRICAO | `[09:25] Diego` |
| ADR-005-TRADEOFF-01 | `docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md` | Trade-off | Deduplicação transferida para o cliente; a plataforma não tem como verificar se ele dedupica | TRANSCRICAO | `[09:25] Sofia` |
| ADR-006 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Decisão | Reuso máximo dos padrões do projeto: AppError, Pino, error middleware, módulos, Zod, prefixo WEBHOOK_ | TRANSCRICAO | `[09:30] Larissa` |
| ADR-006-CTX-01 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Restrição | AppError carrega statusCode, errorCode e details; erros de domínio estendem os HTTP genéricos | CODIGO | `src/shared/errors/app-error.ts` |
| ADR-006-CTX-02 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Restrição | Error middleware já converte AppError, ZodError e erros do Prisma sem precisar de alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-CTX-03 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Restrição | requireRole existente será reaproveitado para exigir role ADMIN no replay de DLQ | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-006-CTX-04 | `docs/adrs/ADR-006-reuso-dos-padroes-existentes.md` | Restrição | Todos os modelos usam UUID em Char(36), padrão que a outbox deve seguir | CODIGO | `prisma/schema.prisma` |
| ADR-007 | `docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md` | Decisão | A linha da outbox guarda o payload já renderizado, como snapshot do estado no momento do evento | TRANSCRICAO | `[09:52] Larissa` |
| ADR-007-ALT-01 | `docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md` | Alternativa considerada | Guardar só order_id e renderizar no envio, descartado por produzir evento internamente incoerente | TRANSCRICAO | `[09:51] Bruno` |
