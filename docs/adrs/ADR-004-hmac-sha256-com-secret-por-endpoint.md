# ADR-004: Autenticação por HMAC-SHA256 com secret única por endpoint e rotação com grace period

## Status

Aceito em 2026-08-05

## Contexto

A feature envia dados de pedidos para endpoints HTTP que estão fora da infraestrutura da
plataforma. Isso levanta dois problemas do lado de quem recebe:

1. Como o cliente sabe que a requisição veio realmente da plataforma, e não de um terceiro
   que descobriu a URL do endpoint dele?
2. Como o cliente sabe que o payload não foi adulterado no caminho?

Não há sessão nem token de usuário nesse fluxo: é a plataforma chamando o cliente, e não o
contrário. O mecanismo de autenticação existente da API, JWT via `authenticate` em
`src/middlewares/auth.middleware.ts`, protege as requisições que chegam, não as que saem.

Há ainda um risco operacional concreto: já houve caso de cliente vazando uma secret em log de
aplicação do próprio lado. Qualquer esquema de credencial precisa considerar que o
comprometimento vai acontecer, e prever a troca.

## Decisão

**Assinatura HMAC-SHA256 sobre o corpo da requisição**, enviada em um header dedicado
(`X-Signature`). O cliente recalcula a assinatura do lado dele com a secret compartilhada e
compara.

**Cada endpoint de webhook tem sua própria secret.** Não existe secret global da plataforma.
A secret é gerada pela plataforma no momento do cadastro e devolvida na resposta de criação.
Ela é armazenada na configuração do webhook, junto com URL, `customer_id` e estado ativo.

**A secret é rotacionável pelo cliente via API.** Quando ele solicita rotação, a secret
antiga permanece válida em paralelo com a nova por **24 horas**, dando tempo de migrar os
sistemas do lado dele. Depois desse grace period, a antiga deixa de valer.

Como consequência direta, durante a janela de rotação a plataforma assina com a secret nova e
o cliente aceita qualquer uma das duas.

## Alternativas Consideradas

### Secret global da plataforma, compartilhada entre todos os endpoints

Uma única chave, mais simples de gerenciar.

**Trade-off que levou ao descarte:** o vazamento de uma secret comprometeria a integração de
todos os clientes ao mesmo tempo. O raio de explosão de um incidente do lado de um único
cliente, que é onde o vazamento historicamente acontece, passaria a ser a base inteira.

### Rotação imediata, sem grace period

Trocar a secret e invalidar a anterior no mesmo instante.

**Trade-off que levou ao descarte:** obrigaria o cliente a fazer o corte de forma
perfeitamente sincronizada com a plataforma. Qualquer defasagem entre a rotação e o deploy do
lado dele derrubaria a validação das notificações recebidas nesse intervalo, transformando um
procedimento de segurança em incidente de integração.

### Algoritmo de hash alternativo ao SHA-256

Outras funções de hash para o HMAC.

**Trade-off que levou ao descarte:** HMAC-SHA256 é o padrão de mercado para assinatura de
webhook, e todo cliente com integração séria já tem biblioteca pronta para verificá-lo.
Escolher outra coisa transferiria custo de implementação para o cliente sem ganho de
segurança correspondente.

## Consequências

### Positivas

- O cliente consegue verificar tanto a origem quanto a integridade do payload, sem precisar
  de infraestrutura adicional.
- O comprometimento de uma secret afeta um único endpoint de um único cliente.
- A rotação com grace period permite trocar a credencial sem janela de indisponibilidade da
  integração, o que torna a rotação um procedimento rotineiro em vez de uma operação de
  risco.
- Padrão amplamente conhecido, o que reduz o esforço de integração do lado do cliente.

### Negativas

- A plataforma passa a armazenar material secreto por endpoint, e esse armazenamento vira
  alvo. Exige cuidado com acesso ao dado e garantia de que a secret nunca apareça em log. O
  logger em `src/shared/logger/index.ts` já redige `authorization`, `*.token`, `*.password` e
  `*.passwordHash`, mas não conhece os campos deste módulo, o que torna a extensão da lista de
  redação uma tarefa obrigatória da implementação, e não um detalhe.
- A janela de 24 horas de rotação é, por definição, um período em que duas credenciais válidas
  coexistem. É uma concessão deliberada de segurança em favor da operabilidade.
- Manter duas secrets ativas em paralelo acrescenta estado e complexidade tanto na modelagem
  quanto na lógica de assinatura.
- A verificação da assinatura é responsabilidade do cliente. A plataforma não tem como saber
  se ele realmente valida, ou se apenas ignora o header.
- A geração de secret e a implementação do HMAC são pontos sensíveis o bastante para exigirem
  revisão de segurança dedicada antes do deploy, o que foi acordado como parte do escopo.

## Referências

- Necessidade de autenticação e integridade: `[09:19] Sofia`
- HMAC como padrão e `X-Signature`: `[09:20] Sofia`
- SHA-256 como algoritmo: `[09:20] Bruno` pergunta, `[09:20] Sofia` responde
- Secret por endpoint: `[09:21] Sofia`
- Campos da configuração de webhook: `[09:21] Bruno`
- Rotação com grace period de 24h: `[09:21] Sofia`, fechada em `[09:22] Sofia`
- Caso real de vazamento de secret: `[09:22] Diego`
- Revisão de segurança antes do deploy: `[09:46] Sofia`
- Logger e redação atual: `src/shared/logger/index.ts`
- Autenticação de entrada existente: `src/middlewares/auth.middleware.ts`
