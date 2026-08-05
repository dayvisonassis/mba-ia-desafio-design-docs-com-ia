# ADR-006: Reuso dos padrões existentes do projeto no módulo de webhooks

## Status

Aceito em 2026-08-05

## Contexto

O sistema de webhooks é a primeira feature da plataforma que introduz processamento
assíncrono e um segundo processo de execução. É natural, diante disso, a tentação de tratá-la
como um subsistema à parte, com convenções próprias.

O código atual, porém, tem padrões estabelecidos e consistentes em todos os cinco módulos
existentes (`auth`, `users`, `customers`, `products`, `orders`):

- **Organização por módulo.** Cada domínio é uma pasta em `src/modules/` com controller,
  service, repository, routes e schemas. O roteador de cada módulo é montado em
  `src/routes/index.ts` por uma função `build...Router`.
- **Hierarquia de erros.** `AppError` em `src/shared/errors/app-error.ts` carrega
  `statusCode`, `errorCode` e `details`. Sobre ela existem erros HTTP genéricos e erros de
  domínio específicos em `src/shared/errors/http-errors.ts`, como `InvalidStatusTransitionError`
  e `InsufficientStockError`.
- **Convenção de código de erro.** Identificadores em `SCREAMING_SNAKE_CASE` que descrevem a
  condição: `INVALID_STATUS_TRANSITION`, `INSUFFICIENT_STOCK`, `INACTIVE_PRODUCT`.
- **Tratamento centralizado de erro.** `src/middlewares/error.middleware.ts` já converte
  `AppError`, `ZodError` e erros conhecidos do Prisma em resposta HTTP no formato
  `{ error: { code, message, details } }`.
- **Validação declarativa.** O middleware `validate` em
  `src/middlewares/validate.middleware.ts` aplica schemas Zod a body, query e params, e
  converte falha em `ValidationError`.
- **Autorização.** `authenticate` e `requireRole` em `src/middlewares/auth.middleware.ts`.
- **Log estruturado.** Pino configurado em `src/shared/logger/index.ts`, com base fixa de
  serviço e ambiente, timestamp ISO e lista de redação de campos sensíveis.
- **Identificadores.** Todos os modelos em `prisma/schema.prisma` usam UUID em `Char(36)`.

A decisão a tomar era se o módulo de webhooks adota essas convenções ou constrói as suas.

## Decisão

**O módulo de webhooks reaproveita ao máximo o que já existe.** Concretamente:

- Pasta nova, a ser criada em `src/modules/webhooks/`, com a mesma estrutura dos demais
  módulos: controller, service, repository, routes e schemas. O roteador é montado no
  `src/routes/index.ts` existente, como os outros.
- A lógica de processamento do worker fica dentro do módulo, em arquivo a ser criado no
  caminho `src/modules/webhooks/webhook.worker.ts`. Apenas a entry-point do processo, a ser
  criada em `src/worker.ts`, fica fora, ao lado do `src/server.ts` existente.
- Os erros do módulo estendem `AppError`, seguindo a hierarquia existente.
- **Todos os códigos de erro do módulo usam o prefixo `WEBHOOK_`**: `WEBHOOK_NOT_FOUND`,
  `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`, e assim por diante.
- Nenhuma mudança em `src/middlewares/error.middleware.ts`. Como os erros novos estendem
  `AppError`, o middleware já os trata sem alteração.
- Validação de entrada por schemas Zod aplicados pelo middleware `validate` existente.
  A exigência de URL `https` no cadastro do webhook é uma validação de schema, não um
  mecanismo novo.
- **Autorização por `requireRole`, já existente.** O endpoint de replay de DLQ exige role
  `ADMIN` e registra quem executou, para auditoria. O restante do CRUD de configuração aceita
  qualquer role autenticada por enquanto.
- Log com o Pino já configurado. Nenhuma biblioteca de log nova.
- Identificadores em UUID, como o resto do projeto.
- O worker instancia seu **próprio `PrismaClient`**, porque o client é por processo. Mesmo
  banco, mesma `DATABASE_URL`, instância separada.

## Alternativas Consideradas

### Tratar webhooks como subsistema com convenções próprias

Modelar o módulo de forma independente, com sua própria hierarquia de erros, seu próprio
formato de log e sua própria estrutura de pastas, justificando pela natureza assíncrona da
feature.

**Trade-off que levou ao descarte:** cria uma segunda maneira de fazer as mesmas coisas
dentro do mesmo repositório. O custo aparece na manutenção: quem trabalha em `orders` e
depois em `webhooks` precisa carregar dois conjuntos de convenções, e cada melhoria em
infraestrutura transversal precisa ser feita duas vezes. A natureza assíncrona do worker não
exige convenções diferentes de nomenclatura, erro ou log.

### Middleware de erro dedicado para o módulo

Um tratamento de erro próprio, específico para as respostas de webhook.

**Trade-off que levou ao descarte:** o middleware central em
`src/middlewares/error.middleware.ts` já resolve o caso pela herança de `AppError`. Um
middleware paralelo duplicaria o formato de resposta de erro, com risco real de os dois
divergirem ao longo do tempo, e a divergência apareceria para o consumidor da API.

## Consequências

### Positivas

- Curva de aprendizado próxima de zero para quem já conhece qualquer outro módulo do
  projeto.
- Nenhuma alteração necessária em infraestrutura compartilhada. O error middleware, o
  validate, o logger e os middlewares de autenticação funcionam para o módulo novo como
  estão.
- Formato de resposta de erro consistente em toda a API, incluindo os endpoints novos.
- O prefixo `WEBHOOK_` torna imediato identificar a origem de um erro em log e em resposta.
- Menos superfície de decisão durante a implementação: as convenções já estão dadas, o
  esforço vai para o problema de domínio.

### Negativas

- O módulo herda as limitações dos padrões atuais junto com os benefícios. Onde a convenção
  existente for inadequada para o caso assíncrono, será preciso conviver com ela ou mudá-la
  para todo mundo.
- A hierarquia de erro é modelada para o ciclo requisição e resposta HTTP. O worker não roda
  dentro de uma requisição, e `statusCode` não significa nada no contexto dele. Usar
  `AppError` no worker carrega um campo sem sentido naquele contexto.
- O logger em `src/shared/logger/index.ts` tem `base: { service: 'order-management-api' }`
  fixo. Os logs do worker vão sair identificados como se fossem da API, o que atrapalha a
  separação em ferramenta de observabilidade. Isso precisa ser resolvido na implementação, e
  é um caso concreto em que o reuso cego custa caro.
- A lista de redação do logger não conhece os campos deste módulo. Secret e assinatura
  precisam ser acrescentados explicitamente, senão o reuso do padrão dá uma falsa sensação de
  proteção.
- Manter o CRUD de configuração aberto a qualquer role autenticada é uma decisão de
  conveniência, assumida como temporária, que deixa uma dívida de autorização registrada.

## Referências

- Estrutura de módulo proposta: `[09:27] Bruno`
- Localização do worker: `[09:28] Bruno`, `[09:28] Diego`
- Padrão de erros e prefixo `WEBHOOK_`: `[09:28] Bruno`, `[09:29] Larissa`
- Pino e error middleware sem alteração: `[09:29] Bruno`
- `PrismaClient` separado por processo: `[09:29] Diego`, `[09:30] Bruno`
- Decisão de reuso máximo: `[09:30] Larissa`
- Role `ADMIN` no replay e reuso do `requireRole`: `[09:36] Sofia`, `[09:36] Larissa`
- CRUD aberto a qualquer role autenticada por enquanto: `[09:36] Marcos`, `[09:37] Sofia`
- Validação de URL `https` como schema Zod: `[09:23] Sofia`
- UUID como padrão: `[09:51] Larissa`
- Código de referência: `src/modules/orders/order.routes.ts`, `src/routes/index.ts`,
  `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts`,
  `src/middlewares/error.middleware.ts`, `src/middlewares/validate.middleware.ts`,
  `src/middlewares/auth.middleware.ts`, `src/shared/logger/index.ts`, `prisma/schema.prisma`
