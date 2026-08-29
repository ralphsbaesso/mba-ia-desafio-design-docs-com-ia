# ADR-006 — Reuso dos padrões existentes do projeto

| | |
| --- | --- |
| **Status** | Aceita |
| **Data** | Reunião de definição da feature (`TRANSCRICAO.md`, quinta-feira, 09:00–09:53) |
| **Decisores** | Bruno (Eng. Pleno, Pedidos — condutor do bloco), Larissa (Tech Lead), Diego (Eng. Sênior), Sofia (Eng. de Segurança) |
| **RFC** | [RFC — Entrega de Notificações de Pedido por Webhook](../RFC.md), §3.4 |
| **Relacionadas** | Todos os demais ADRs deste pacote — é a decisão que define *como* eles são implementados |

> Este é o ADR que **referencia explicitamente o código existente**. Todos os caminhos e símbolos da
> coluna "Onde está no código" foram verificados no repositório; caminhos marcados como **a criar** são
> propostas desta feature e ainda não existem.

## Contexto

O OMS tem convenções estabelecidas e consistentes em todos os seus domínios. Bruno abriu o bloco de
estrutura de código descrevendo-as: "A gente tem um padrão claro na codebase. Cada domínio é um módulo em
src/modules com controller, service, repository, routes e schemas" (`[09:27] Bruno`).

A feature de webhooks é grande o bastante para tentar a introdução de ferramental próprio — uma
biblioteca de filas, um logger específico do worker, uma hierarquia de erros paralela. A decisão a tomar
era se a feature seguiria as convenções da casa ou abriria exceções.

O contexto de prazo pesa: a entrega é estimada em três sprints com data negociada com um cliente que
ameaçou sair (`[09:45] Marcos`, `[09:47] Larissa`). Divergir dos padrões custa tempo de implementação, de
revisão e de manutenção futura.

## Decisão

**Reuso máximo dos padrões existentes; nenhuma dependência, convenção ou ferramenta nova.** Fechada em
`[09:30] Larissa`: "Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de
módulos, padrão de schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros."

| O que se reusa | Onde está no código | Como a feature o estende |
| --- | --- | --- |
| **Estrutura de módulo** — `routes` / `controller` / `service` / `repository` / `schemas` | `src/modules/orders/`, e idêntico em `customers/`, `products/`, `users/` | Um módulo **a criar** em `src/modules/webhooks/`, com os mesmos arquivos. "Vou propor uma pasta src/modules/webhooks com toda a estrutura" (`[09:27] Bruno`) |
| **Hierarquia de erros** | `src/shared/errors/app-error.ts:3` (`AppError(message, statusCode, errorCode, details)`) e as subclasses em `src/shared/errors/http-errors.ts` (`NotFoundError:27`, `ConflictError:33`, `InvalidStatusTransitionError:45`, `InsufficientStockError:55`) | Novas subclasses de `AppError` com códigos de prefixo `WEBHOOK_`, exportadas pelo barrel `src/shared/errors/index.ts`. "Tem classe AppError, classes específicas tipo InsufficientStockError… Quero seguir igual pra webhook" (`[09:28] Bruno`) |
| **Padrão de código de erro** | `INSUFFICIENT_STOCK` (`http-errors.ts:59`), `INVALID_STATUS_TRANSITION` (`http-errors.ts:49`) | `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` e demais. "Prefixo WEBHOOK_ pra tudo do módulo" (`[09:29] Larissa`) |
| **Middleware de erro centralizado** | `src/middlewares/error.middleware.ts:14` — responde `{ error: { code, message, details? } }`, já trata `AppError`, `ZodError` e `PrismaClientKnownRequestError` | **Nenhuma alteração.** Herdando de `AppError`, os erros do módulo são tratados automaticamente: "Vai pegar nossos erros sem precisar mudar nada" (`[09:29] Bruno`) |
| **Validação por schema** | `src/middlewares/validate.middleware.ts:11` — `validate({ body, query, params })` com Zod | Schemas Zod por rota, incluindo a exigência de `https` na URL do webhook — que Sofia classificou como "só uma validação no schema Zod" (`[09:23] Sofia`) |
| **Autorização por papel** | `src/middlewares/auth.middleware.ts:49` — `requireRole(...roles)`; hoje usado uma única vez, em `src/modules/users/user.routes.ts:15` | `requireRole('ADMIN')` no endpoint de replay da dead letter. "a gente reaproveita o requireRole que já existe" (`[09:36] Larissa`) |
| **Logger** | `src/shared/logger/index.ts` — Pino com `redact` de campos sensíveis (linhas 4–11) | Mesmo logger, sem nada novo: "o logger, que é Pino, já tá no projeto inteiro. Não vamos botar nada novo" (`[09:29] Bruno`). A lista de `redact` precisa ganhar o campo da secret — ver [ADR-003](./ADR-003-hmac-sha256-com-secret-por-endpoint.md) |
| **Registro de rotas e injeção de dependência** | `src/routes/index.ts:21` (`buildApiRouter`) e `src/app.ts:26` (`buildControllers`, com o encadeamento repository → service → controller) | Um router `/webhooks` registrado no mesmo ponto, e o módulo montado no mesmo encadeamento |
| **Convenções de modelagem** | `prisma/schema.prisma` — `@id @default(uuid()) @db.Char(36)`, `@@map` em snake_case, valores em centavos | Tabelas novas seguindo as mesmas convenções. "UUID, segue o padrão do resto do projeto. Tudo é uuid" (`[09:51] Larissa`) |
| **Formato de listagem paginada** | `src/shared/http/response.ts:22` — `paginated(data, page, pageSize, total)` | Listagem de webhooks e de histórico de entregas no mesmo formato |
| **Client de banco** | `src/config/database.ts:4` — `createPrismaClient()` | O worker cria a própria instância pelo mesmo caminho (`[09:30] Bruno`) |

A única extensão estrutural é a **segunda entry-point**, ao lado de `src/server.ts`, para o processo do
worker (`[09:11] Larissa`) — e ela existe por exigência de [ADR-005](./ADR-005-worker-separado-em-polling.md),
não por divergência de padrão.

## Alternativas Consideradas

### A. Ferramental próprio para o módulo de webhooks

Introduzir uma biblioteca de filas/agendamento, um logger dedicado ao worker ou uma hierarquia de erros
independente, sob o argumento de que o domínio é suficientemente distinto dos demais.

**Rejeitada.** Bruno recusou explicitamente a parte de logging — "Não vamos botar nada novo"
(`[09:29] Bruno`) — e Larissa generalizou a postura para toda a feature (`[09:30] Larissa`). O custo seria
duplo: tempo de implementação dentro de um prazo já apertado, e uma segunda convenção que todo mantenedor
futuro do OMS precisaria aprender.

### B. Injetar o repositório de webhooks dentro do `OrderService`

Caminho convencional de composição para acoplar o novo módulo ao existente.

**Rejeitada** em favor de uma função que recebe a transação em curso — `publishWebhookEvent(tx, order,
fromStatus, toStatus)`. Bruno propôs (`[09:41] Bruno`) e Diego endossou: "Boa, função pura recebendo o
tx. Não precisa injetar repository inteiro" (`[09:41] Diego`). Isso mantém o `OrderService`
(`src/modules/orders/order.service.ts:126`) sem uma dependência nova no construtor e preserva o controle
transacional onde ele já vive.

## Consequências

### Positivas

- **Velocidade de entrega.** Nenhum tempo gasto escolhendo, integrando ou aprendendo ferramental novo —
  decisivo diante das três sprints estimadas.
- **Integração gratuita com o tratamento de erros.** Como os erros herdam de `AppError`, o middleware
  central já os formata corretamente, sem uma linha de alteração
  (`src/middlewares/error.middleware.ts:14`).
- **Previsibilidade para quem mantém.** Quem conhece o módulo de pedidos consegue navegar o de webhooks
  sem contexto adicional.
- **Superfície de dependências inalterada.** Nada novo a auditar, atualizar ou monitorar por
  vulnerabilidade.
- **Consistência de contrato da API.** Erros, paginação e validação seguem o formato que os clientes já
  consomem nos demais endpoints.

### Negativas

- **A feature herda as limitações dos padrões atuais.** O que estiver ausente hoje precisa ser
  estendido — o caso concreto é a lista de `redact` do logger, que cobre `*.token` e `*.password` mas não
  o campo da secret (`src/shared/logger/index.ts`, linhas 4–11): sem essa extensão, a decisão de
  [ADR-003](./ADR-003-hmac-sha256-com-secret-por-endpoint.md) fica exposta nos nossos próprios logs.
- **O padrão de módulo não cobre processos de background.** A estrutura
  `routes/controller/service/repository` foi desenhada para o ciclo requisição-resposta; o worker não tem
  controller nem rotas, e seu arquivo de processamento é uma peça sem precedente no projeto — inclusive o
  nome ficou indefinido na reunião (`webhook.worker.ts` ou `webhook.processor.ts`, `[09:28] Bruno`).
- **Acoplamento assumido a decisões antigas.** Mudanças futuras no formato de erro ou no logger passam a
  ter alcance maior, incluindo o caminho de entrega de webhooks.
- **Risco de forçar o encaixe.** Onde o domínio de webhooks divergir de fato do padrão CRUD, seguir a
  convenção pode produzir código menos natural do que uma abordagem própria produziria.
