# 06 — Mapa do código (âncoras VERIFICADAS)

Recorte: toda menção a arquivo, classe, método ou padrão do código existente feita na reunião,
**confirmada com `grep`/`ls`** no repositório antes de entrar aqui. Menção na reunião não é prova
de existência — a última seção lista o que **não existe** e por isso só pode ser citado como
proposta.

Base da seção "Integração com o sistema existente" do FDD.

## Âncoras que existem

| # | Âncora (`arquivo:linha`) | Símbolo | Para que serve | Menção na reunião |
| --- | --- | --- | --- | --- |
| COD-01 | `src/modules/orders/order.service.ts:126` | `OrderService.changeStatus(id, input, userId)` | Todo o corpo roda dentro de `this.prisma.$transaction` (linha 131): valida transição, debita/repõe estoque, `tx.order.update` (156) e `tx.orderStatusHistory.create` (159). **Ponto de inserção do evento na outbox, na mesma transação.** | `[09:40] Bruno` |
| COD-02 | `src/modules/orders/order.service.ts:131` | `this.prisma.$transaction(async (tx) => …)` | A transação cujo `tx` seria passado a `publishWebhookEvent(tx, …)`. | `[09:41] Bruno`; `[09:41] Diego` |
| COD-03 | `src/modules/orders/order.service.ts:159` | `tx.orderStatusHistory.create` | Insert de auditoria já feito na transação — precedente direto do insert na outbox. | `[09:04] Bruno`; `[09:06] Diego` |
| COD-04 | `src/modules/orders/order.service.ts:152,155` | `debitStock` / `replenishStock` (privados, definidos em `:204` e `:233`) | Decremento/incremento de `stockQuantity` dentro da mesma transação — é o "peso" que Bruno usa para descartar o síncrono. | `[09:04] Bruno` |
| COD-05 | `src/modules/orders/order.status.ts` | `canTransition:12`, `allowedTransitions:16`, `isTerminal:20`, `shouldDebitStock:29`, `shouldReplenishStock:33`; mapa `transitions:3` | Máquina de estados: define quais transições existem e, portanto, quais eventos `order.status_changed` podem ser gerados. Transições válidas: `PENDING→PAID|CANCELLED`, `PAID→PROCESSING|CANCELLED`, `PROCESSING→SHIPPED|CANCELLED`, `SHIPPED→DELIVERED`; `DELIVERED` e `CANCELLED` são terminais. | `[09:12] Larissa` (sequência PAID→PROCESSING→SHIPPED) |
| COD-06 | `src/shared/errors/app-error.ts:3` | `class AppError(message, statusCode, errorCode, details)` | Base de todos os erros; os códigos `WEBHOOK_*` devem estender daqui. | `[09:28] Bruno` |
| COD-07 | `src/shared/errors/http-errors.ts` | `BadRequestError:3`, `ValidationError:9`, `UnauthorizedError:15`, `ForbiddenError:21`, `NotFoundError:27`, `ConflictError:33`, `UnprocessableEntityError:39`, `InvalidStatusTransitionError:45`, `InsufficientStockError:55` | Subclasses e o padrão de código em SCREAMING_SNAKE (`INVALID_STATUS_TRANSITION:49`, `INSUFFICIENT_STOCK:59`) que os `WEBHOOK_*` replicam. | `[09:28] Bruno` (cita `AppError`, `InsufficientStockError`, `InvalidStatusTransitionError`, `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) |
| COD-08 | `src/shared/errors/index.ts` | Barrel de exportação dos erros | Ponto de registro das novas classes de erro do módulo de webhooks. | implícito em `[09:28] Bruno` |
| COD-09 | `src/middlewares/error.middleware.ts:14` | `errorMiddleware` | Handler central. Formato `{ error: { code, message, details? } }` (`:16`); trata `AppError` (`:15`), `ZodError` (`:26`) e `PrismaClientKnownRequestError` P2002/P2025 (`:37`,`:48`). **Pega os erros `WEBHOOK_*` sem alteração.** | `[09:29] Bruno` ("não precisa mudar nada") |
| COD-10 | `src/middlewares/auth.middleware.ts:27` | `authenticate` | JWT Bearer; popula `req.user = { id, email, role }` (`:42`). Confirma o ponto de Bruno: o JWT é do usuário operador do sistema, não do cliente — daí `customer_id` vir no body/path. | `[09:32] Bruno`; `[09:32] Larissa` |
| COD-11 | `src/middlewares/auth.middleware.ts:49` | `requireRole(...roles)` | Guard de role; `AuthUser['role']` é `'ADMIN' \| 'OPERATOR'` (`:9`). É o que o endpoint de replay reaproveita. | `[09:36] Larissa` |
| COD-12 | `src/modules/users/user.routes.ts:15` | `requireRole('ADMIN')` | **Único uso atual** de `requireRole('ADMIN')` no projeto — precedente para o guard do endpoint de replay. | `[09:36] Sofia`; `[09:36] Larissa` |
| COD-13 | `src/middlewares/validate.middleware.ts:11` | `validate({ body, query, params })` | Validação Zod por rota; converte `ZodError` em `ValidationError` (`:31`). É onde entra a validação de `https` na URL do webhook. | `[09:23] Sofia` ("só uma validação no schema Zod"); `[09:30] Larissa` (padrão de schemas Zod) |
| COD-14 | `src/shared/logger/index.ts:14` | `createLogger()` / `logger:32` (Pino) | Logger único do projeto, com `redact` (`:4`–`:11`) de `*.password`, `*.passwordHash`, `*.token`, `*.accessToken`, `req.headers.authorization`, `req.headers.cookie`. **Relevante para o tratamento da secret HMAC** — hoje `*.secret` NÃO está na lista de redact. | `[09:29] Bruno` ("o logger, que é Pino, já tá no projeto inteiro") |
| COD-15 | `src/middlewares/request-logger.middleware.ts:5` | `requestLogger` | `X-Request-Id` (header de entrada ou uuid, `:6`–`:8`) e log `http_request` com `durationMs` (`:14`–`:24`). Base da observabilidade dos endpoints novos. | padrão do projeto (`[09:30] Larissa`, reuso máximo) |
| COD-16 | `src/routes/index.ts:21` | `buildApiRouter(controllers)` | Registra os routers sob `/api/v1` (`src/app.ts:67`); é onde um `router.use('/webhooks', …)` se pluga. O type `Controllers:13` também precisa ganhar a entrada. | `[09:27] Bruno` (módulo com routes) |
| COD-17 | `src/app.ts:26` | `buildControllers(prisma)` | DI manual repository → service → controller (ex.: orders em `:42`–`:44`). O módulo de webhooks segue o mesmo encadeamento. | `[09:27] Bruno` |
| COD-18 | `src/app.ts:55` | `buildApp(deps)` | Monta o Express: `express.json({ limit: '1mb' })` (`:59`), `requestLogger` (`:60`), `/health` (`:62`), rota 404 (`:69`) e `errorMiddleware` (`:73`). | — |
| COD-19 | `src/server.ts:6` | `bootstrap()` | Entry-point da API: `buildApp`, `listen`, shutdown em SIGINT/SIGTERM com `prisma.$disconnect()` (`:13`–`:21`). **Modelo para o `src/worker.ts` proposto.** | `[09:11] Larissa` ("tipo o que a gente já tem em src/server.ts") |
| COD-20 | `src/config/database.ts:4,10` | `createPrismaClient()` / `prisma` | Instância única exportada; o worker, sendo outro processo, cria a sua própria via `createPrismaClient()`. | `[09:29] Diego`; `[09:30] Bruno` |
| COD-21 | `src/config/env.ts:3` | `envSchema` (Zod) | Config validada: `NODE_ENV`, `PORT`, `LOG_LEVEL`, `DATABASE_URL`, `JWT_SECRET`, `JWT_EXPIRES_IN`. Onde entrariam parâmetros do worker (intervalo de polling, timeout). | `[09:30] Bruno` ("mesma DATABASE_URL") |
| COD-22 | `prisma/schema.prisma` | Convenções | `@id @default(uuid()) @db.Char(36)` (ex.: `:26`, `:75`, `:117`), `@@map` em snake_case (`:37`, `:96`, `:130`), valores monetários em centavos `Int` (`priceCents:61`, `totalCents:81`). **Confirma a decisão de UUID na outbox.** | `[09:51] Larissa` ("tudo é uuid") |
| COD-23 | `prisma/schema.prisma:116` | `model OrderStatusHistory` → `@@map("order_status_history")` | Tabela de auditoria com `fromStatus`/`toStatus`, `changedAt`, índices em `orderId` e `changedAt` (`:128`,`:129`). **Precedente de modelagem para `webhook_outbox` e `webhook_dead_letter`**, inclusive o padrão de índices que Diego descreveu. | `[09:06] Diego`; `[09:08] Diego` |
| COD-24 | `prisma/schema.prisma:74` | `model Order` → `@@map("orders")` | Campos que compõem o payload: `orderNumber:76`, `customerId:77`, `status:78`, `totalCents:81`. Índices existentes em `status:93` e `createdAt:94` — mesmo padrão que a outbox vai usar. | `[09:43] Diego` (campos do payload) |
| COD-25 | `prisma/schema.prisma:16` | `enum OrderStatus` | `PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED` — o domínio válido da lista de status assinados por um webhook. | `[09:33] Marcos` ("só quero saber quando vira SHIPPED e DELIVERED") |
| COD-26 | `prisma/schema.prisma:40` | `model Customer` → `@@map("customers")` | Alvo da FK `customer_id` na configuração de webhook. | `[09:21] Bruno` |
| COD-27 | Padrão de módulo — `src/modules/orders/` | `order.routes.ts`, `order.controller.ts`, `order.service.ts`, `order.repository.ts`, `order.schemas.ts`, `order.status.ts` | Estrutura a replicar em `src/modules/webhooks/`. Verificado também em `customers/`, `products/`, `users/`. | `[09:27] Bruno` |
| COD-28 | `src/modules/orders/order.routes.ts:12` | `buildOrderRouter(controller)` | Modelo do router: `router.use(authenticate)` (`:14`) e `validate({...})` por rota; note `PATCH /:id/status` (`:19`) — o endpoint que dispara o evento. | `[09:33] Bruno` (PATCH/DELETE/GET) |
| COD-29 | `src/shared/http/response.ts:22` | `paginated(data, page, pageSize, total)` | Formato de listagem (`{ data, pagination }`) usado por orders (`order.service.ts:41`), products e customers. Formato dos endpoints de listagem de webhooks e de deliveries. | `[09:34] Marcos` ("últimos 100 webhooks") |
| COD-30 | `tests/helpers/factories.ts` | `getTestApp:10`, `createTestUser:17`, `loginAndGetToken:33`, `createTestCustomer:42`, `createTestProduct:62`, `bootstrapAuthenticatedUser:76` | Infra de teste (Vitest + Supertest, ver `tests/orders.test.ts`) a reutilizar nos testes ponta a ponta previstos na estimativa. | `[09:46] Larissa` ("testes ponta a ponta") |
| COD-31 | `package.json` (bloco `scripts`) | `dev`, `build`, `start`, `db:migrate`, `db:seed`, `test`, `lint` | Onde entraria o `npm run worker` proposto. Stack confirmada: Node ≥20, `@prisma/client` 5.22.0, `express` 4.21.1, `pino` 9.5.0, `zod` 3.23.8, `uuid` 11.0.3. | `[09:11] Larissa` |

## Mencionado na reunião mas NÃO existe hoje (é proposta, não âncora)

Citar apenas como "a ser criado". Nenhum destes pode ser referenciado como código existente.

| Item citado | Localização | Verificação |
| --- | --- | --- |
| `src/worker.ts` | `[09:11] Larissa`; `[09:28] Bruno` | Não existe (`find src -type f` não lista). |
| script `npm run worker` | `[09:11] Larissa` | Ausente do bloco `scripts` de `package.json`. |
| `src/modules/webhooks/` (e `webhook.worker.ts` / `webhook.processor.ts`) | `[09:27] Bruno`; `[09:28] Bruno` | Diretório inexistente. |
| tabela `webhook_outbox` | `[09:06] Diego` | Ausente de `prisma/schema.prisma`. |
| tabela `webhook_dead_letter` | `[09:18] Diego` | Ausente de `prisma/schema.prisma`. |
| `publishWebhookEvent(tx, order, fromStatus, toStatus)` | `[09:41] Bruno` | Símbolo inexistente. |
| códigos `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED` | `[09:28] Bruno` | Nenhum código `WEBHOOK_*` em `src/`. |
| endpoints `/webhooks`, `/webhooks/:id/deliveries`, `/admin/webhooks/dead-letter/:id/replay` | `[09:34] Marcos`; `[09:35] Diego` | Não registrados em `src/routes/index.ts`. |
| trigger de banco | `[09:09] Bruno` | Nenhuma trigger em `prisma/migrations/`; e descartado (DESC-03). |

## Observações relevantes para o FDD

1. **`express.json({ limit: '1mb' })`** (`src/app.ts:59`) já limita o corpo das requisições de
   entrada; o teto de 64KB (NUM-16) é do **payload de saída** do evento — controle distinto, a ser
   implementado no worker/na inserção.
2. **O `redact` do Pino** (`src/shared/logger/index.ts:4`) cobre `*.token`/`*.password` mas **não**
   `*.secret`. Estender a lista é um ponto concreto de integração para a decisão de HMAC.
3. **`changeStatus` recebe `userId`** (`order.service.ts:129`) e o grava em `changedById` — o mesmo
   padrão de atribuição serve à exigência de auditoria do replay (`[09:36] Sofia`).
4. **`OrderStatusHistory` guarda `fromStatus`/`toStatus`** exatamente como o payload do evento
   (`[09:43] Diego`), o que torna a outbox um espelho natural dessa tabela.
