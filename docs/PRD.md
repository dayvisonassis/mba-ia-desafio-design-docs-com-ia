# PRD: Order Management System — Sistema de Webhooks de Notificação de Pedidos

| | |
| --- | --- |
| **Versão** | 1.0 |
| **Data** | 2026-08-05 |
| **Responsável** | Marcos (Product Manager) |
| **Documentos relacionados** | [RFC](RFC.md), [FDD](FDD.md), [ADRs](adrs/README.md) |

---

## Resumo

O Order Management System passa a notificar automaticamente os clientes B2B integrados sempre
que o status de um pedido deles muda. Em vez de consultar a API periodicamente para descobrir
se algo mudou, o cliente cadastra um endereço próprio e recebe a informação assim que a
mudança acontece.

A feature nasce de um pedido formal de três clientes que hoje sustentam integrações caras e
lentas baseadas em consulta repetida. Ela cobre o cadastro e a gestão dos endereços de
notificação, o envio autenticado das notificações, a repetição automática em caso de falha do
destinatário e a consulta ao histórico de entregas.

---

## Contexto e problema

**Público-alvo**

- **Clientes B2B integrados por API.** Atlas Comercial, MaxDistribuição e Nova Cargo hoje;
  qualquer cliente com integração no futuro.
- **Equipes de integração desses clientes**, que constroem e mantêm o consumo da API.
- **Operadores da plataforma com papel administrativo**, responsáveis por reprocessar
  notificações que falharam em definitivo.

**Cenários de uso**

- Um pedido da Atlas passa de `PROCESSING` para `SHIPPED`. O sistema da Atlas é avisado em
  segundos e dispara a comunicação com o cliente final, sem ter perguntado nada à plataforma.
- A MaxDistribuição só se interessa por dois status e configura o endereço para receber apenas
  esses, sem tráfego irrelevante.
- A Nova Cargo passa por uma manutenção planejada de duas horas. Ao voltar, recebe as
  notificações do período, porque o sistema continuou tentando.
- A equipe de integração da Atlas suspeita de notificação não recebida e consulta o histórico
  de entregas para verificar o que foi enviado, quando e com qual resultado.
- Um endereço fica fora do ar por tempo demais e as notificações esgotam as tentativas. Um
  operador administrativo reprocessa manualmente depois que o cliente confirma a normalização.

**Onde a feature será implantada**

No Order Management System existente, uma aplicação Node.js com TypeScript e MySQL. A feature
acrescenta um módulo à API atual e um processo novo de execução em segundo plano. Nenhum
sistema novo é criado, e nenhum contrato existente é alterado.

**Problemas priorizados**

- **A integração por consulta repetida é lenta e cara para o cliente.** Os três clientes
  consultam a listagem de pedidos periodicamente só para descobrir se algo mudou. O custo é do
  lado deles, e a informação chega com o atraso do intervalo de consulta. *Prioridade alta.*
- **Risco comercial concreto.** A Atlas Comercial sinalizou que pode migrar para um concorrente
  se a entrega não sair até o fim do trimestre. *Prioridade alta.*
- **A plataforma não tem nenhum mecanismo de notificação externa.** Não existe evento, fila ou
  webhook em lugar nenhum do sistema. Não é uma lacuna deste caso de uso; é uma capacidade
  ausente. *Prioridade alta.*
- **Sem histórico de entrega, a depuração de integração é adivinhação.** Quando um cliente diz
  que não recebeu, não há como verificar. *Prioridade média.*

---

## Objetivos e métricas

| Objetivo | Métrica | Meta |
| --- | --- | --- |
| **Notificar** a mudança de status em tempo próximo do real | Latência p95 entre a mudança de status e a primeira tentativa de entrega | Abaixo de **10 segundos** |
| **Eliminar** a necessidade de consulta repetida pelos clientes integrados | Chamadas à listagem de pedidos originadas dos três clientes | Redução de pelo menos **80%** em 60 dias após a integração *(hipótese: a meta não foi definida na reunião)* |
| **Não perder** nenhuma notificação | Eventos gerados sem entrega confirmada e sem registro de falha | **Zero** |
| **Absorver** indisponibilidade planejada do cliente | Janela coberta pelas tentativas automáticas | Aproximadamente **15 horas** entre a primeira falha e a última tentativa |
| **Reter** os três clientes que originaram o pedido | Clientes integrados e recebendo notificações | **3 de 3** até o fim de novembro |
| **Entregar** dentro da capacidade planejada | Sprints consumidas, incluindo revisão de segurança | **3 sprints** |

---

## Escopo

**Incluso**

- Cadastro, consulta, edição e remoção dos endereços de notificação do cliente
- Escolha, por endereço, de quais status de pedido geram notificação
- Envio automático da notificação quando o status muda
- Autenticação da notificação, de forma que o cliente possa comprovar a origem e a
  integridade do conteúdo
- Rotação da credencial de assinatura pelo cliente, sem interrupção da integração
- Repetição automática em caso de falha do destinatário, com espaçamento crescente
- Registro em separado das notificações que esgotaram as tentativas
- Consulta ao histórico de entregas de um endereço
- Reprocessamento manual, restrito a operadores administrativos

**Fora de escopo**

- **Notificação por email ao cliente quando o webhook dele está falhando.** Levantada na
  reunião e explicitamente adiada: entra em fase futura, depois que o impacto real for medido.
- **Dashboard visual para o cliente acompanhar seus webhooks.** Descartada para esta fase:
  é projeto separado do time de frontend. Nesta entrega o cliente interage apenas por API.
- **Rate limiting das notificações enviadas a um mesmo cliente.** Levantada e deixada em
  aberto: será observada em produção e implementada se virar problema.
- **Arquivamento das notificações já entregues.** Reconhecida como necessária e colocada fora
  do escopo desta feature.
- **Webhooks de entrada**, do cliente para a plataforma. O escopo é apenas de saída: os
  clientes querem receber, não enviar.
- **Processamento paralelo das notificações.** A ordem de entrega por pedido depende de o
  processamento ser sequencial, e o paralelismo fica para quando a escala exigir.

---

## Requisitos funcionais

### RF-01 Cadastro de endereço de notificação

O cliente registra um endereço que passará a receber as notificações dos pedidos dele.

**Fluxo principal**
1. O usuário autenticado informa o cliente dono do endereço, a URL de destino e a lista de
   status que deseja receber.
2. O sistema valida a URL e a lista de status.
3. O sistema gera a credencial de assinatura e cria o cadastro como ativo.
4. O sistema devolve o cadastro criado junto com a credencial, exibida apenas nesta resposta.

**Fluxos alternativos e exceções**
- O cliente informado não existe: o cadastro é recusado.
- O mesmo cliente pode ter mais de um endereço cadastrado, com filtros diferentes.

**Erros previstos**
- URL ausente, malformada ou sem canal seguro.
- Lista de status vazia ou com valor que não corresponde a um status de pedido válido.

**Prioridade:** alta

### RF-02 Consulta dos endereços de um cliente

O usuário lista os endereços cadastrados de um cliente, com o filtro de status e o estado de
cada um.

**Fluxo principal**
1. O usuário autenticado consulta os endereços de um cliente.
2. O sistema devolve a lista paginada.

**Fluxos alternativos e exceções**
- Cliente sem nenhum endereço cadastrado: lista vazia, não erro.

**Erros previstos**
- Cliente não informado ou identificador inválido.

**Prioridade:** alta

### RF-03 Edição de endereço de notificação

O cliente altera a URL, o filtro de status ou o estado ativo de um endereço já cadastrado.

**Fluxo principal**
1. O usuário autenticado informa o endereço e os campos a alterar.
2. O sistema valida e aplica apenas os campos informados.
3. O sistema devolve o cadastro atualizado.

**Fluxos alternativos e exceções**
- Desativar um endereço não cancela as notificações já geradas para ele; elas são encerradas
  sem tentativa de envio quando chega a vez de processá-las.
- A credencial de assinatura não é alterada por esta operação.

**Erros previstos**
- Endereço inexistente.
- URL ou lista de status inválidas.

**Prioridade:** alta

### RF-04 Remoção de endereço de notificação

O cliente remove um endereço que não quer mais usar.

**Fluxo principal**
1. O usuário autenticado solicita a remoção do endereço.
2. O sistema remove o cadastro e confirma.

**Fluxos alternativos e exceções**
- O histórico de entregas do endereço removido é preservado para auditoria.
- Notificações pendentes para o endereço removido são encerradas sem tentativa de envio.

**Erros previstos**
- Endereço inexistente.

**Prioridade:** média

### RF-05 Filtro de notificações por status

Cada endereço recebe apenas os status que declarou querer.

**Fluxo principal**
1. No cadastro ou na edição, o cliente declara a lista de status de interesse.
2. Quando o status de um pedido muda, o sistema considera apenas os endereços do cliente dono
   do pedido cujo filtro inclui o status de destino.
3. Se nenhum endereço se interessa por aquele status, nenhuma notificação é gerada.

**Fluxos alternativos e exceções**
- O filtro é aplicado na geração da notificação, não no envio, para não gerar trabalho que
  seria descartado depois.

**Erros previstos**
- Status informado que não corresponde a nenhum estado válido de pedido.

**Prioridade:** alta

### RF-06 Notificação automática na mudança de status

Toda mudança de status de pedido gera notificação para os endereços interessados.

**Fluxo principal**
1. O status de um pedido muda por meio da API.
2. O sistema registra a notificação como parte da mesma operação que altera o status.
3. O envio ocorre logo em seguida, de forma assíncrona.
4. O conteúdo enviado descreve o pedido como ele estava no momento da transição, e não como
   está no momento do envio.

**Fluxos alternativos e exceções**
- Se a mudança de status falhar por qualquer motivo, a notificação não é gerada.
- Se o registro da notificação falhar, a mudança de status inteira é desfeita. Não existe caso
  de status alterado sem notificação registrada.

**Erros previstos**
- Conteúdo da notificação acima do limite de tamanho definido.

**Prioridade:** alta

### RF-07 Autenticação da notificação enviada

O cliente consegue comprovar que a notificação veio da plataforma e que o conteúdo não foi
alterado no caminho.

**Fluxo principal**
1. Antes de enviar, o sistema assina o conteúdo com a credencial exclusiva daquele endereço.
2. A assinatura acompanha a notificação.
3. O cliente recalcula a assinatura do lado dele e compara.

**Fluxos alternativos e exceções**
- Cada endereço tem sua própria credencial. Não existe credencial única da plataforma.
- A notificação também carrega o instante do envio, permitindo ao cliente detectar reenvio
  malicioso.

**Erros previstos**
- Endereço sem credencial ativa, situação que indica inconsistência e é registrada como tal.

**Prioridade:** alta

### RF-08 Rotação da credencial de assinatura

O cliente troca a credencial sem interromper a integração.

**Fluxo principal**
1. O cliente solicita a rotação para um endereço.
2. O sistema gera uma credencial nova e a devolve, exibida apenas nesta resposta.
3. A credencial anterior permanece válida por 24 horas, para o cliente migrar seus sistemas.
4. Encerrado o prazo, a anterior deixa de valer.

**Fluxos alternativos e exceções**
- Nova rotação solicitada dentro do prazo de graça é recusada, para não invalidar uma
  credencial que o cliente ainda pode estar usando.

**Erros previstos**
- Endereço inexistente.
- Rotação já em andamento.

**Prioridade:** alta

### RF-09 Repetição automática em caso de falha

Uma notificação que não chega ao destino é tentada novamente, com espaçamento crescente.

**Fluxo principal**
1. O sistema considera falha a resposta fora da faixa de sucesso, o esgotamento do tempo de
   espera ou o erro de conexão.
2. Uma nova tentativa é agendada, com intervalo maior a cada vez.
3. São no máximo cinco tentativas, distribuídas ao longo de aproximadamente 15 horas.

**Fluxos alternativos e exceções**
- Se o endereço foi desativado ou removido entre uma tentativa e outra, a notificação é
  encerrada sem novas tentativas.

**Erros previstos**
- Destino não responde no tempo previsto.
- Destino responde com erro.
- Destino inacessível por falha de rede ou de certificado.

**Prioridade:** alta

### RF-10 Registro das notificações que falharam em definitivo

Notificações que esgotam as tentativas ficam registradas em separado, com o motivo.

**Fluxo principal**
1. Esgotada a última tentativa, o sistema move a notificação para um registro dedicado de
   falhas definitivas.
2. O registro guarda o conteúdo original, o endereço de destino e o motivo da última falha.

**Fluxos alternativos e exceções**
- Nenhuma notificação é descartada em silêncio. Ela está entregue, aguardando tentativa ou
  registrada como falha definitiva.

**Erros previstos**
- Não se aplica: este é o destino de erro dos demais fluxos.

**Prioridade:** alta

### RF-11 Consulta do histórico de entregas

O cliente verifica o que a plataforma enviou, quando e com que resultado.

**Fluxo principal**
1. O usuário autenticado consulta o histórico de um endereço.
2. O sistema devolve as entregas mais recentes primeiro, com o conteúdo enviado, o resultado,
   o número da tentativa, a resposta recebida e o tempo de resposta.

**Fluxos alternativos e exceções**
- O histórico inclui tanto tentativas bem-sucedidas quanto falhas, para que a depuração
  enxergue a sequência completa.

**Erros previstos**
- Endereço inexistente.

**Prioridade:** média

### RF-12 Reprocessamento manual de falha definitiva

Um operador administrativo reenvia uma notificação que falhou em definitivo.

**Fluxo principal**
1. O operador com papel administrativo solicita o reprocessamento de um registro de falha.
2. O sistema recoloca a notificação na fila de envio, com o mesmo identificador de evento e a
   contagem de tentativas zerada.
3. O sistema registra quem executou o reprocessamento e quando.

**Fluxos alternativos e exceções**
- Usuário sem papel administrativo tem a operação recusada.
- Registro já reprocessado tem a operação recusada, para não duplicar a notificação.

**Erros previstos**
- Registro de falha inexistente.
- Registro já reprocessado.
- Permissão insuficiente.

**Prioridade:** média

---

## Requisitos não funcionais

**Performance**
- Latência p95 abaixo de 10 segundos entre a mudança de status e a primeira tentativa de
  entrega.
- Piso de latência de 2 segundos, decorrente do intervalo de verificação do processo de envio.
- Tempo máximo de espera por resposta do destinatário: 10 segundos.

**Disponibilidade**
- 99,9% de disponibilidade mensal do serviço de notificação *(hipótese: a meta não foi
  definida na reunião)*.
- A indisponibilidade do processo de envio não pode impedir a mudança de status de pedidos: as
  notificações se acumulam e são entregues quando ele retorna.

**Segurança e autorização**
- Toda notificação enviada é assinada com HMAC-SHA256.
- Cada endereço tem credencial exclusiva. Não existe credencial global da plataforma.
- Credencial rotacionável, com 24 horas de validade paralela da anterior.
- O destino precisa usar canal seguro. Endereço sem TLS é recusado na validação.
- O reprocessamento manual exige papel administrativo e registra o autor, para auditoria.
- A credencial nunca aparece em log nem em resposta de leitura.

**Observabilidade**
- Log estruturado de cada tentativa de entrega, com resultado e duração.
- Correlação entre a requisição que mudou o status e a entrega correspondente.
- Métricas que permitam detectar o processo de envio parado, incluindo a idade da notificação
  pendente mais antiga.
- Alerta para entrada no registro de falhas definitivas.

**Confiabilidade e integridade de dados**
- O registro da notificação e a mudança de status são atômicos: ou os dois acontecem, ou
  nenhum dos dois.
- Garantia de entrega ao menos uma vez. A entrega repetida é possível e prevista, e cada
  notificação carrega identificador único e estável para o cliente descartar duplicatas.
- Ordem de entrega preservada por pedido enquanto o processamento for sequencial.
- Conteúdo da notificação limitado a 64KB. Acima disso o sistema retorna erro em vez de
  truncar, porque um conteúdo desse tamanho indica defeito.

**Compatibilidade e portabilidade**
- Nenhum contrato existente da API é alterado.
- Os endpoints novos entram sob o mesmo versionamento da API atual.
- Nenhuma dependência nova de execução é introduzida para o núcleo da feature.

**Compliance**
- Trilha de auditoria do reprocessamento manual, com autor e momento.
- Histórico de entregas preservado mesmo após a remoção do endereço.

---

## Decisões e trade-offs

### Decisão: registrar a notificação na mesma transação que altera o status (padrão Outbox)

- **Justificativa:** garante por construção que não existe status alterado sem notificação
  registrada, nem notificação de uma mudança que não aconteceu.
- **Trade-off:** a operação central de mudança de status ganha mais uma escrita, e um defeito
  no módulo de notificação passa a conseguir bloqueá-la.
- **Registro:** [ADR-001](adrs/ADR-001-outbox-no-mysql.md)

### Decisão: envio por processo separado, verificando periodicamente as notificações pendentes

- **Justificativa:** isola falhas entre a API e a entrega, e usa apenas a infraestrutura que já
  existe.
- **Trade-off:** cria um piso de latência de 2 segundos e um processo novo para operar e
  monitorar. Se ele cair, nada falha visivelmente, mas nada é entregue.
- **Registro:** [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)

### Decisão: cinco tentativas com espaçamento crescente, depois registro de falha definitiva

- **Justificativa:** cobre indisponibilidades reais de cliente, incluindo manutenções longas,
  sem intervenção humana, e ainda assim tem um fim.
- **Trade-off:** uma notificação pode levar até cerca de 15 horas para chegar em cenário de
  falha prolongada, e o reprocessamento depois disso é manual.
- **Registro:** [ADR-003](adrs/ADR-003-retry-com-backoff-e-dlq.md)

### Decisão: assinatura HMAC-SHA256 com credencial exclusiva por endereço

- **Justificativa:** o vazamento de uma credencial compromete um único endereço de um único
  cliente, e não a base inteira.
- **Trade-off:** a plataforma passa a armazenar e proteger material secreto por endereço, e a
  janela de rotação mantém duas credenciais válidas ao mesmo tempo por 24 horas.
- **Registro:** [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md)

### Decisão: entrega ao menos uma vez, com deduplicação pelo cliente

- **Justificativa:** garantir entrega exatamente uma vez exigiria coordenação entre os dois
  lados e complexidade desproporcional. É o modelo adotado pelo mercado.
- **Trade-off:** transfere responsabilidade ao cliente. Um cliente que ignore o identificador
  de evento processa duplicatas, e a plataforma não tem como verificar se ele dedupica.
- **Registro:** [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)

### Decisão: reaproveitar os padrões já estabelecidos no projeto

- **Justificativa:** curva de aprendizado próxima de zero, consistência de comportamento e
  nenhuma alteração em infraestrutura compartilhada.
- **Trade-off:** herda também as limitações desses padrões, que em dois pontos precisam ser
  ajustados para servir a um processo que não roda dentro de uma requisição HTTP.
- **Registro:** [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)

### Decisão: guardar o conteúdo da notificação no momento em que o evento acontece

- **Justificativa:** o envio pode ocorrer horas depois. Congelar o conteúdo mantém a
  notificação internamente coerente: a transição descrita e os dados correspondem ao mesmo
  instante.
- **Trade-off:** duplica dado, e uma correção posterior no pedido não se reflete em
  notificações já geradas.
- **Registro:** [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)

---

## Dependências

### Organizacional: revisão de segurança antes do deploy

A engenheira de segurança pediu no mínimo dois dias úteis para revisar o código de geração de
credencial e de assinatura antes da subida. A revisão faz parte da estimativa de três sprints
e é bloqueante para o lançamento.

### Organizacional: documentação de integração no portal do desenvolvedor

O comportamento de entrega repetida e a necessidade de deduplicação precisam estar
documentados de forma destacada para os clientes antes da liberação. Sem isso, o modelo de
entrega ao menos uma vez vira defeito percebido em vez de contrato conhecido. O Product
Manager assumiu essa entrega.

### Externa: implementação do lado do cliente

Cada cliente precisa expor um endereço com canal seguro, verificar a assinatura recebida e
descartar notificações repetidas pelo identificador de evento. A plataforma não tem como
verificar se isso foi feito.

### Técnica: definição das ferramentas de observabilidade

Não existe hoje no projeto biblioteca de métricas nem de rastreamento distribuído. A escolha
precisa ser feita para que os requisitos de observabilidade sejam atendíveis. O núcleo
funcional da feature não depende disso.

---

## Riscos e mitigação

### O processo de envio para de funcionar e ninguém percebe

- **Probabilidade:** média
- **Impacto:** alto. Os clientes deixam de ser notificados e nenhum indicador atual da API
  acusa problema, porque nenhuma requisição HTTP falha.
- **Mitigação:**
  - Métrica da idade da notificação pendente mais antiga, com alerta em limite baixo
  - Verificação de saúde do próprio processo de envio
  - Registro periódico de atividade, permitindo detectar ausência
- **Plano de contingência:** reiniciar o processo. Nada se perde, porque o estado das
  notificações vive no banco, e a fila drena sozinha.

### Um defeito no módulo de notificação bloqueia a mudança de status de pedidos

- **Probabilidade:** baixa
- **Impacto:** alto. A operação central do sistema fica indisponível, afetando todos os
  clientes e não apenas os integrados.
- **Mitigação:**
  - Manter mínima a lógica executada dentro da transação: sem chamada externa e sem regra
    complexa
  - Teste de integração cobrindo o caminho transacional com falha injetada
  - Revisão dedicada do trecho acrescentado à operação existente
- **Plano de contingência:** desligar a geração de notificações por configuração de ambiente,
  aceitando perder eventos no período, para restaurar a operação de pedidos.

### Vazamento de credencial de assinatura

- **Probabilidade:** média
- **Impacto:** alto. Um terceiro passa a forjar notificações válidas para aquele endereço. Já
  houve caso real de cliente expondo credencial no próprio log.
- **Mitigação:**
  - Credencial exclusiva por endereço, o que limita o alcance do incidente
  - Rotação com prazo de graça, tornando a troca um procedimento sem indisponibilidade
  - Proibição explícita de registrar a credencial em log, dos dois lados da fronteira
  - Revisão de segurança dedicada antes do lançamento
- **Plano de contingência:** rotacionar a credencial do endereço afetado e comunicar o cliente.

### O cliente não implementa deduplicação e processa notificações repetidas

- **Probabilidade:** média
- **Impacto:** médio. O efeito colateral acontece no sistema do cliente e pode gerar dado
  duplicado, dependendo do que ele faz com a notificação.
- **Mitigação:**
  - Identificador de evento estável entre tentativas e preservado no reprocessamento
  - Documentação destacada no portal do desenvolvedor
- **Plano de contingência:** nenhum do lado da plataforma. É consequência assumida do modelo de
  entrega escolhido.

### O prazo de três sprints não se sustenta

- **Probabilidade:** média
- **Impacto:** alto. Existe risco comercial declarado de perda de um dos três clientes.
- **Mitigação:**
  - Escopo mantido enxuto, com seis itens explicitamente fora desta fase
  - Revisão de segurança agendada com antecedência, em vez de descoberta no fim
  - Sequência de implementação que entrega primeiro o núcleo de geração e envio
- **Plano de contingência:** negociar com o cliente a entrega em duas etapas, priorizando o
  envio automático e adiando o histórico de entregas e o reprocessamento manual.

---

## Critérios de aceitação

- Um cliente consegue cadastrar um endereço de notificação e recebe a credencial de assinatura
  na resposta do cadastro.
- Endereço sem canal seguro é recusado no cadastro, com erro de validação.
- Toda mudança de status de um pedido gera notificação para cada endereço ativo do cliente cujo
  filtro inclua o status de destino, e para nenhum outro.
- Mudança de status sem nenhum endereço interessado não gera notificação alguma.
- Se a mudança de status é desfeita, nenhuma notificação correspondente permanece registrada.
- Se o registro da notificação falha, a mudança de status é desfeita por completo, incluindo o
  histórico de status e a movimentação de estoque.
- A notificação chega ao endereço do cliente em até 10 segundos após a mudança de status, no
  cenário em que o destino responde normalmente.
- O conteúdo recebido descreve o pedido como ele estava no momento da transição, mesmo que o
  pedido tenha mudado depois.
- O cliente consegue validar a assinatura recebida usando a credencial daquele endereço.
- Um destino que responde com erro é tentado exatamente cinco vezes, com intervalos crescentes,
  antes de ser considerado falha definitiva.
- Toda tentativa da mesma notificação carrega o mesmo identificador de evento.
- Uma notificação que esgota as tentativas está registrada como falha definitiva, com o
  conteúdo original e o motivo da última falha.
- Nenhuma notificação gerada termina sem estar entregue, aguardando tentativa ou registrada
  como falha definitiva.
- Após a rotação, a credencial anterior continua válida por exatamente 24 horas.
- O reprocessamento manual é recusado para usuário sem papel administrativo, e registra o autor
  quando executado.
- O histórico de entregas apresenta tentativas bem-sucedidas e falhas, com resultado e tempo de
  resposta.
- Nenhum registro de log de nenhum dos dois processos contém credencial ou assinatura.
- Nenhum endpoint existente da API teve seu contrato alterado.

---

## Testes e validação

**Tipos de teste obrigatórios**

- **Testes de integração do caminho transacional**, com falha injetada no registro da
  notificação, verificando que a mudança de status, o histórico e o estoque são desfeitos por
  completo. É o teste de maior valor da feature, porque cobre a garantia que justifica toda a
  arquitetura escolhida.
- **Testes de integração dos endpoints de gestão**, cobrindo cadastro, consulta, edição,
  remoção, rotação e reprocessamento, incluindo os casos de recusa. Seguem o padrão dos testes
  atuais em `tests/`, com Vitest e Supertest sobre banco de teste.
- **Testes unitários da política de repetição**, verificando a progressão dos intervalos, o
  limite de cinco tentativas e a transição para falha definitiva.
- **Testes unitários da assinatura**, verificando que a assinatura gerada confere com o
  cálculo independente e que a credencial anterior continua aceita dentro do prazo de graça.
- **Testes de autorização**, verificando que o reprocessamento exige papel administrativo.
- **Revisão de segurança manual**, conduzida pela engenheira de segurança, focada na geração de
  credencial e na implementação da assinatura. Bloqueante para o lançamento.

**Estratégia de validação**

- Desenvolvimento orientado a teste na lógica de repetição e de assinatura, onde a regra é
  determinística e o custo de erro é alto.
- Validação exploratória com um destino controlado que simula os cenários de falha: resposta de
  erro, ausência de resposta dentro do tempo limite e indisponibilidade prolongada.
- Verificação ponta a ponta antes do lançamento, com um dos três clientes em ambiente de
  homologação, confirmando que ele consegue validar a assinatura e deduplicar pelo
  identificador de evento.
- Acompanhamento das métricas de latência e de falhas definitivas nos primeiros 30 dias, que
  também alimenta as decisões adiadas sobre limitação de envio e notificação por email.
