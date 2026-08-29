# FDD (TDD) — Sistema de Webhooks de Notificação de Pedidos

| | |
| --- | --- |
| **Documento** | Feature Design Document — equivalente de mercado: **Technical Design Document / tech spec** |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Status** | Pronto para implementação — sujeito a revisão de segurança bloqueante antes do deploy |
| **Origem** | Reunião de definição (`TRANSCRICAO.md`, quinta-feira, 09:00–09:53) |
| **Relacionados** | [PRD](./PRD.md) · [RFC](./RFC.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md) |

> **Nota de nomenclatura.** O enunciado do desafio chama este documento de *FDD (Feature Design
> Document)*. No mercado, "FDD" costuma designar *Feature-Driven Development* ou *Functional Design
> Document* — este último descreve o sistema pela ótica de negócio, o oposto do que aqui se pede. O
> documento abaixo é uma **tech spec**: o que será construído e como. O *porquê* de cada decisão está nos
> [ADRs](./adrs/) e **não é reargumentado aqui** — este documento referencia e implementa.

---

## 1. Contexto e motivação técnica

O OMS é uma API Express + TypeScript sobre MySQL via Prisma, com módulos de autenticação, usuários,
clientes, produtos e pedidos. **Não existe hoje nenhum mecanismo de eventos, filas, agendamento ou
chamada de saída** — não há biblioteca de fila no `package.json`, nenhum processo além de
`src/server.ts`, e nenhuma tabela de eventos em `prisma/schema.prisma`.

O ciclo de vida do pedido é governado por uma máquina de estados explícita
(`src/modules/orders/order.status.ts`) e toda mudança de status passa por um único método,
`OrderService.changeStatus` (`src/modules/orders/order.service.ts:126`), cujo corpo inteiro roda dentro
de `prisma.$transaction` (linha 131). Esse método é o **único ponto de entrada** de mudança de status no
sistema — o que torna a integração da feature localizada e verificável.

A tarefa técnica é acoplar a esse ponto um mecanismo de notificação externa que satisfaça três exigências
simultâneas e em tensão: **nenhum evento pode ser perdido** quando o status muda
([ADR-001](./adrs/ADR-001-outbox-no-mysql.md)); **nenhuma chamada externa pode entrar no caminho crítico**
da transação (`[09:04] Bruno`); e **nenhuma infraestrutura nova** pode ser introduzida
([ADR-001](./adrs/ADR-001-outbox-no-mysql.md), alternativa B).

## 2. Objetivos técnicos

| ID | Objetivo | Verificação |
| --- | --- | --- |
| **OT-01** | Registrar o evento de mudança de status na mesma transação que já atualiza `orders` e `order_status_history`, sem nenhuma I/O externa no caminho crítico. | Teste de integração cobrindo commit e rollback |
| **OT-02** | Entregar o evento ao endpoint do cliente em **menos de 10 segundos** no caminho feliz, com piso arquitetural de 2 segundos. | Medição da métrica `webhook_delivery_lag_seconds` |
| **OT-03** | Executar a política de retentativa 1m/5m/30m/2h/12h com teto de 5 tentativas e destino terminal em dead letter. | Teste do cálculo de agendamento + teste contra destino indisponível |
| **OT-04** | Assinar cada envio com HMAC-SHA256 usando a secret do endpoint de destino, sem jamais registrar a secret em log. | Teste de verificação da assinatura + inspeção do `redact` do Pino |
| **OT-05** | Expor a API de configuração e consulta seguindo os padrões de rota, validação, erro e paginação já existentes. | Revisão de código contra [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| **OT-06** | Rodar o consumo em processo separado, com ciclo de vida independente da API. | `npm run worker` sobe sem a API; reinício da API não interrompe entregas |

## 3. Escopo e exclusões

> **Convenção de caminhos neste documento.** Todo caminho de arquivo citado sem ressalva **existe hoje
> no repositório** e foi verificado. Os caminhos **a criar** por esta feature são exatamente três, e
> aparecem sempre marcados como novos: `src/modules/webhooks/` (o módulo), `src/worker.ts` (a
> entry-point do worker) e as 4 tabelas da §4.1 em `prisma/schema.prisma`.

### Incluso

Modelagem das tabelas de configuração, outbox, entregas e dead letter; módulo `src/modules/webhooks/`;
extensão de `changeStatus`; entry-point e loop do worker; assinatura HMAC e ciclo de vida da secret; os 7
endpoints da §5; matriz de erros `WEBHOOK_*`; métricas, logs e correlação da §8.

### Excluído

Além dos itens de produto listados no [PRD §5.2](./PRD.md), estas exclusões são **técnicas**:

| Exclusão | Razão | Origem |
| --- | --- | --- |
| Rotina de arquivamento/expurgo das tabelas | Reconhecida como necessária e colocada fora do escopo da feature | `[09:08] Diego` |
| Rate limiting de saída por destino | Em aberto; observar antes de implementar | `[09:38] Diego`, `[09:39] Larissa` |
| Paralelização do worker (partição por pedido, lock pessimista) | Adiada; a instância única é premissa da ordenação | `[09:13] Diego` |
| Alerta ativo de webhook degradado | Fora desta fase | `[09:37] Larissa` |
| Distributed tracing com propagação de contexto entre API e worker | Não discutido na reunião; ver §8.3 para o que é feito no lugar | — |

## 4. Fluxos detalhados

### 4.1 Modelo de dados

Quatro tabelas novas, seguindo as convenções verificadas em `prisma/schema.prisma`: chave `@id
@default(uuid()) @db.Char(36)`, `@@map` em snake_case, valores monetários em centavos como `Int`
([ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes.md)).

| Tabela | Papel | Campos essenciais | Índices |
| --- | --- | --- | --- |
| `webhook_endpoints` | Configuração do cliente | `id`, `customerId` (FK → `customers`), `url`, `secret`, `previousSecret`, `previousSecretExpiresAt`, `subscribedStatuses`, `active`, `createdAt`, `updatedAt` | `customerId`, `active` |
| `webhook_outbox` | Fila de trabalho | `id`, `endpointId` (FK), `orderId`, `eventType`, `payload` (snapshot renderizado), `status` (`PENDING`/`PROCESSING`/`FAILED`/`DELIVERED`), `attempts`, `nextAttemptAt`, `createdAt` | `(status, nextAttemptAt)`, `createdAt`, `orderId` |
| `webhook_deliveries` | Histórico de tentativas | `id`, `outboxId`, `endpointId`, `attempt`, `requestBody`, `responseStatus`, `responseBody`, `durationMs`, `error`, `createdAt` | `endpointId`, `outboxId`, `createdAt` |
| `webhook_dead_letter` | Destino terminal | `id`, `outboxId`, `endpointId`, `payload`, `failureReason`, `attempts`, `failedAt`, `replayedAt`, `replayedById` (FK → `users`) | `endpointId`, `failedAt` |

Os campos de configuração vêm de `[09:21] Bruno` (url, secret, customer, ativo) e `[09:21] Sofia`
(rotação); os estados da outbox de `[09:08] Diego`; os campos de `webhook_deliveries` de `[09:34] Marcos`
("sucesso/falha, payload, response, tempo de resposta"); os de `webhook_dead_letter` de `[09:18] Diego`
("a payload, motivo da falha e timestamp") e `[09:36] Sofia` (autoria do replay).

### 4.2 Fluxo A — criação do evento na outbox (síncrono, dentro da transação)

```
PATCH /api/v1/orders/:id/status
        │
        ▼  OrderService.changeStatus  (order.service.ts:126)
┌──────────────────── prisma.$transaction ────────────────────┐
│ 1. carrega o pedido com itens                               │
│ 2. valida a transição      → canTransition (order.status.ts)│
│ 3. movimenta estoque       → debitStock / replenishStock    │
│ 4. tx.order.update                                          │
│ 5. tx.orderStatusHistory.create                             │
│ 6. ► publishWebhookEvent(tx, order, fromStatus, toStatus)   │
│      a. busca endpoints ativos do customerId que assinam    │
│         toStatus                        ← ADR-008           │
│      b. se nenhum → retorna sem gravar                      │
│      c. renderiza o payload (snapshot) ← ADR-007            │
│      d. valida tamanho ≤ 64 KB → excede: WEBHOOK_PAYLOAD_   │
│         TOO_LARGE (aborta a transação)                      │
│      e. insere 1 linha em webhook_outbox por endpoint,      │
│         status=PENDING, attempts=0, nextAttemptAt=now()     │
└─────────────────────────────────────────────────────────────┘
        │ commit  → pedido alterado E evento(s) registrado(s)
        │ rollback → nada mudou (ADR-001)
```

`publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o **client transacional**, não o
`PrismaClient` — assinatura decidida em `[09:41] Bruno` e endossada em `[09:41] Diego` justamente para
não injetar o repositório de webhooks no `OrderService`.

**Payload do evento** (`[09:43] Diego`), gerado no passo (c):

```json
{
  "event_id": "9f1c3d2e-8b7a-4c15-9e0d-6a2f4b8c1d3e",
  "event_type": "order.status_changed",
  "timestamp": "2026-08-29T14:32:11.482Z",
  "order_id": "3b7e1a94-5c22-4d61-8f03-9a7c2e5b1d40",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "c81a4f66-2d90-4b7e-a5c3-71e0f9d82b45",
  "total_cents": 149900
}
```

A composição de itens **não** é enviada: "Não manda items pra não inflar. Se o cliente quiser detalhes,
ele bate no GET /orders/:id depois" (`[09:43] Diego`).

### 4.3 Fluxo B — processamento pelo worker

```
src/worker.ts  (entry-point nova, ao lado de src/server.ts)
  └─ createPrismaClient()          ← instância própria (config/database.ts:4)
  └─ loop a cada 2 s (ADR-005):

     1. SELECT em webhook_outbox
          WHERE status IN ('PENDING','FAILED') AND nextAttemptAt <= now()
          ORDER BY createdAt ASC
          LIMIT <batch pequeno>                       ← [09:08] Diego
     2. marca o lote como PROCESSING
     3. para cada evento, sequencialmente:
          a. resolve o endpoint e a secret vigente
          b. assina: HMAC-SHA256(secret, corpo cru)   ← ADR-003
          c. POST no endpoint, timeout 10 s           ← [09:42] Diego
          d. grava webhook_deliveries (tentativa, status, corpo, duração)
          e. 2xx        → status=DELIVERED
             demais/erro → attempts++ , §4.4
```

Processamento **sequencial dentro do lote** e **instância única**: é o que preserva a ordem por pedido
([ADR-005](./adrs/ADR-005-worker-separado-em-polling.md)).

**Requisição enviada ao cliente** (headers de `[09:44] Diego` e `[09:44] Sofia`):

```http
POST /webhooks/oms HTTP/1.1
Host: cliente.exemplo.com
Content-Type: application/json
X-Event-Id: 9f1c3d2e-8b7a-4c15-9e0d-6a2f4b8c1d3e
X-Webhook-Id: 1d4b7c90-3e58-4a26-b0f1-8c9d2e6a5b73
X-Signature: sha256=7b2a...c91f
X-Timestamp: 2026-08-29T14:32:13.104Z

{ ...payload da §4.2... }
```

- `X-Event-Id` — UUID do evento, **estável entre todas as tentativas**; é a chave de deduplicação do
  cliente ([ADR-004](./adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)).
- `X-Signature` — HMAC-SHA256 do **corpo cru**, exatamente como transmitido.
- `X-Timestamp` — momento **do envio**, não do evento: existe para o cliente detectar replay attack se
  quiser (`[09:44] Diego`). Numa retentativa, difere do `timestamp` do payload.
- `X-Webhook-Id` — identifica qual cadastro originou o envio, para clientes com vários endpoints
  (`[09:44] Sofia`).

### 4.4 Fluxo C — retentativa com backoff

```
falha na tentativa N (N = attempts após incremento)
   │
   ├─ N < 5 → status = FAILED
   │          nextAttemptAt = now() + BACKOFF[N-1]
   │          BACKOFF = [1min, 5min, 30min, 2h, 12h]     ← [09:17] Diego
   │          (volta ao pool do worker no próximo ciclo elegível)
   │
   └─ N = 5 → §4.5 (dead letter)
```

Conta como falha: status HTTP fora de 2xx, timeout de 10 s, erro de conexão/DNS e falha de TLS. O
espaçamento total entre a primeira falha e a última tentativa é de aproximadamente **15 horas**.

### 4.5 Fluxo D — dead letter e replay

```
5ª falha
   └─ INSERT em webhook_dead_letter (payload, failureReason, attempts, failedAt)
      UPDATE webhook_outbox SET status='FAILED'   (terminal; nextAttemptAt = NULL)

POST /api/v1/admin/webhooks/dead-letter/:id/replay      ← requireRole('ADMIN')
   └─ nova linha em webhook_outbox: status=PENDING, attempts=0,
      nextAttemptAt=now(), mesmo payload e mesmo event_id
   └─ webhook_dead_letter.replayedAt / replayedById = req.user.id   ← [09:36] Sofia
```

O `event_id` é preservado no replay: se a entrega original chegou ao cliente antes de a resposta se
perder, a deduplicação dele ainda funciona
([ADR-004](./adrs/ADR-004-entrega-at-least-once-com-x-event-id.md)).

### 4.6 Fluxo E — rotação de secret

```
POST /api/v1/webhooks/:id/rotate-secret
   └─ previousSecret          = secret atual
      previousSecretExpiresAt = now() + 24 h        ← [09:21] Sofia
      secret                  = nova secret gerada
   └─ responde a nova secret (única exibição)

durante a janela: assina-se com a secret NOVA; a antiga permanece registrada para
que o cliente, que ainda pode ter a antiga configurada, consiga validar envios
já em trânsito. Expirada a janela, previousSecret é limpa.
```

## 5. Contratos públicos

Todos os endpoints ficam sob o prefixo `/api/v1` já montado em `src/app.ts:67`, registrados via
`buildApiRouter` (`src/routes/index.ts:21`). Todos exigem `authenticate`
(`src/middlewares/auth.middleware.ts:27`); o replay exige adicionalmente `requireRole('ADMIN')`
(linha 49). Erros seguem o formato `{ error: { code, message, details? } }` produzido por
`src/middlewares/error.middleware.ts:14`.

### 5.1 `POST /api/v1/webhooks` — cadastrar endpoint

Origem: `[09:31] Marcos`, `[09:32] Larissa` (customerId no corpo, não no JWT).

**Request**
```json
{
  "customerId": "c81a4f66-2d90-4b7e-a5c3-71e0f9d82b45",
  "url": "https://cliente.exemplo.com/webhooks/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"]
}
```

**Response `201 Created`** — a `secret` é exibida **apenas aqui**:
```json
{
  "id": "1d4b7c90-3e58-4a26-b0f1-8c9d2e6a5b73",
  "customerId": "c81a4f66-2d90-4b7e-a5c3-71e0f9d82b45",
  "url": "https://cliente.exemplo.com/webhooks/oms",
  "subscribedStatuses": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9c1f4a7e2b8d5063af1e94c7d2b60853",
  "createdAt": "2026-08-29T14:20:00.000Z"
}
```

| Status | Quando |
| --- | --- |
| `201` | Criado |
| `400` | `VALIDATION_ERROR` (Zod) ou `WEBHOOK_INVALID_URL` (URL não `https`) |
| `401` | Sem token válido |
| `404` | `WEBHOOK_CUSTOMER_NOT_FOUND` |
| `409` | `WEBHOOK_DUPLICATE_URL` — mesma URL já cadastrada para o cliente |

### 5.2 `GET /api/v1/webhooks?customerId=…&page=&pageSize=` — listar

Origem: `[09:33] Bruno`. Formato de `src/shared/http/response.ts:22`.

**Response `200 OK`** — a `secret` **nunca** aparece em listagem ou leitura:
```json
{
  "data": [
    {
      "id": "1d4b7c90-3e58-4a26-b0f1-8c9d2e6a5b73",
      "customerId": "c81a4f66-2d90-4b7e-a5c3-71e0f9d82b45",
      "url": "https://cliente.exemplo.com/webhooks/oms",
      "subscribedStatuses": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-08-29T14:20:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

| Status | Quando |
| --- | --- |
| `200` | OK (lista vazia inclusive) |
| `400` | `VALIDATION_ERROR` — paginação ou `customerId` inválidos |
| `401` | Sem token válido |

### 5.3 `PATCH /api/v1/webhooks/:id` — alterar

Origem: `[09:33] Bruno`. Campos aceitos: `url`, `subscribedStatuses`, `active`. A `secret` **não** é
alterável por aqui — para isso existe §5.5.

**Request**
```json
{ "subscribedStatuses": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

**Response `200 OK`**: mesmo corpo da §5.2, sem `secret`.

| Status | Quando |
| --- | --- |
| `200` | Atualizado |
| `400` | `VALIDATION_ERROR` ou `WEBHOOK_INVALID_URL` |
| `401` | Sem token válido |
| `404` | `WEBHOOK_NOT_FOUND` |

### 5.4 `DELETE /api/v1/webhooks/:id` — remover

Origem: `[09:33] Bruno`.

**Response `204 No Content`** (sem corpo). Eventos já na outbox para esse endpoint são cancelados; o
histórico de entregas é preservado.

| Status | Quando |
| --- | --- |
| `204` | Removido |
| `401` | Sem token válido |
| `404` | `WEBHOOK_NOT_FOUND` |

### 5.5 `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret

Origem: `[09:21] Sofia`.

**Request**: sem corpo.

**Response `200 OK`**:
```json
{
  "id": "1d4b7c90-3e58-4a26-b0f1-8c9d2e6a5b73",
  "secret": "whsec_4e8b2d61c9f70a3e5d1b8c42a67f9013",
  "previousSecretExpiresAt": "2026-08-30T14:45:00.000Z"
}
```

| Status | Quando |
| --- | --- |
| `200` | Rotacionada |
| `401` | Sem token válido |
| `404` | `WEBHOOK_NOT_FOUND` |
| `409` | `WEBHOOK_ROTATION_IN_PROGRESS` — já existe janela de convivência aberta |

### 5.6 `GET /api/v1/webhooks/:id/deliveries?page=&pageSize=&status=` — histórico de entregas

Origem: `[09:34] Marcos` — "sucesso/falha, payload, response, tempo de resposta".

**Response `200 OK`**:
```json
{
  "data": [
    {
      "id": "5a9e7c31-4d02-4b88-9f6a-2c17e8b3d940",
      "outboxId": "8c2f1b45-9a63-4e07-b1d8-3f6c5a209e74",
      "eventId": "9f1c3d2e-8b7a-4c15-9e0d-6a2f4b8c1d3e",
      "attempt": 2,
      "success": false,
      "responseStatus": 503,
      "responseBody": "service unavailable",
      "durationMs": 1042,
      "error": null,
      "createdAt": "2026-08-29T14:37:15.220Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 137, "totalPages": 7 }
}
```

| Status | Quando |
| --- | --- |
| `200` | OK |
| `400` | `VALIDATION_ERROR` |
| `401` | Sem token válido |
| `404` | `WEBHOOK_NOT_FOUND` |

### 5.7 `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — reprocessar evento morto

Origem: `[09:35] Diego` (o caminho), `[09:36] Sofia` e `[09:36] Larissa` (ADMIN + auditoria).

**Request**: sem corpo.

**Response `202 Accepted`**:
```json
{
  "deadLetterId": "b3f19d47-6c02-4a85-9e13-7d5a2c8f0416",
  "outboxId": "e7c40a92-1b68-4d35-a09f-5c2e8b13d764",
  "eventId": "9f1c3d2e-8b7a-4c15-9e0d-6a2f4b8c1d3e",
  "status": "PENDING",
  "replayedAt": "2026-08-29T16:02:44.318Z",
  "replayedById": "0a5c8e17-93b2-4f60-8d41-6e7a2b9c3f58"
}
```

| Status | Quando |
| --- | --- |
| `202` | Reenfileirado |
| `401` | Sem token válido |
| `403` | `FORBIDDEN` — papel diferente de `ADMIN` (`requireRole`) |
| `404` | `WEBHOOK_DEAD_LETTER_NOT_FOUND` |
| `409` | `WEBHOOK_ALREADY_REPLAYED` |

## 6. Matriz de erros `WEBHOOK_*`

Todas as classes estendem `AppError` (`src/shared/errors/app-error.ts:3`) pelas subclasses HTTP de
`src/shared/errors/http-errors.ts`, são exportadas pelo barrel `src/shared/errors/index.ts` e são
formatadas **sem nenhuma alteração** no middleware central (`src/middlewares/error.middleware.ts:14`) —
`[09:29] Bruno`, [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes.md).

A coluna **Origem** distingue o que foi nomeado na reunião do que é derivado de um comportamento decidido.

| Código | HTTP | Classe base | Quando ocorre | Origem |
| --- | --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | `NotFoundError` | Endpoint inexistente em `webhook_endpoints` | **Literal** — `[09:28] Bruno` |
| `WEBHOOK_INVALID_URL` | 400 | `BadRequestError` | URL ausente, malformada ou não `https` | **Literal** — `[09:28] Bruno`; regra em `[09:23] Sofia` |
| `WEBHOOK_SECRET_REQUIRED` | 400 | `BadRequestError` | Operação de assinatura sem secret vigente para o endpoint | **Literal** — `[09:28] Bruno` |
| `WEBHOOK_CUSTOMER_NOT_FOUND` | 404 | `NotFoundError` | `customerId` do corpo não existe em `customers` | Derivado de `[09:32] Larissa` |
| `WEBHOOK_DUPLICATE_URL` | 409 | `ConflictError` | Mesma URL já cadastrada para o cliente | Derivado de `[09:21] Bruno` |
| `WEBHOOK_INVALID_STATUS_FILTER` | 400 | `BadRequestError` | `subscribedStatuses` vazio ou fora do enum `OrderStatus` | Derivado de `[09:33] Marcos` |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | `UnprocessableEntityError` | Payload renderizado excede 64 KB — **não envia, não trunca** | Derivado de `[09:23] Sofia`, `[09:24] Diego` |
| `WEBHOOK_ROTATION_IN_PROGRESS` | 409 | `ConflictError` | Rotação pedida com janela de 24 h ainda aberta | Derivado de `[09:21] Sofia` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | `NotFoundError` | Registro de dead letter inexistente | Derivado de `[09:35] Diego` |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | `ConflictError` | Registro de dead letter já reprocessado | Derivado de `[09:18] Diego` |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | erro interno do worker | Destino não respondeu em 10 s; não vira resposta HTTP, é gravado em `webhook_deliveries.error` | Derivado de `[09:42] Diego` |
| `WEBHOOK_DELIVERY_FAILED` | — | erro interno do worker | Resposta fora de 2xx, erro de conexão, DNS ou TLS | Derivado de `[09:15] Diego` |

`403 FORBIDDEN` e `401 UNAUTHORIZED` no replay **não** ganham código próprio: vêm de `ForbiddenError` e
`UnauthorizedError` já existentes (`http-errors.ts:21` e `:15`), acionados por `requireRole`.

## 7. Estratégias de resiliência

| Aspecto | Estratégia | Origem |
| --- | --- | --- |
| **Timeout de saída** | 10 s por tentativa; estouro é falha e entra no ciclo de retentativa. | `[09:42] Diego` |
| **Retentativa** | 5 tentativas, backoff 1m/5m/30m/2h/12h, janela total ~15 h. | `[09:17] Diego` |
| **Estado terminal** | Após a 5ª falha, `webhook_dead_letter`; sem retentativa automática. | `[09:18] Diego` |
| **Fallback** | **Não existe canal alternativo** — o e-mail de aviso ficou fora desta fase. A recuperação é o replay manual (§5.7). | `[09:37] Larissa` |
| **Atomicidade** | Evento e mudança de status commitam ou revertem juntos. | `[09:40] Bruno` |
| **Proteção do caminho crítico** | Nenhuma I/O externa dentro da transação: só escrita local ao banco. | `[09:04] Bruno` |
| **Idempotência** | `X-Event-Id` estável entre tentativas e preservado no replay; dedup no cliente. | `[09:25] Diego` |
| **Contenção de carga** | Leitura em lotes pequenos, ordenada por `createdAt`, com índice `(status, nextAttemptAt)`. | `[09:08] Diego` |
| **Encerramento gracioso** | SIGINT/SIGTERM: termina o lote em curso, devolve os não processados a `PENDING` e chama `prisma.$disconnect()`, espelhando `src/server.ts:13`. | Derivado de `[09:11] Diego` |
| **Eventos órfãos** | Linhas travadas em `PROCESSING` além de um limiar voltam a `PENDING` no ciclo seguinte — cobre a queda abrupta do worker. | Derivado de `[09:11] Diego` |
| **Limite de tamanho** | 64 KB verificados **na inserção**, antes do commit: falha cedo em vez de acumular evento inentregável. | `[09:24] Diego` |

**Sem rate limiting de saída** nesta fase — decisão consciente, registrada como questão aberta
([RFC §5, RFC-Q-01](./RFC.md)).

## 8. Observabilidade

### 8.1 Métricas

| Métrica | Tipo | Labels | Para que serve |
| --- | --- | --- | --- |
| `webhook_outbox_pending_total` | gauge | — | Saúde da fila; crescimento sustentado indica worker parado ou saturado |
| `webhook_delivery_lag_seconds` | histogram | — | Latência entre `createdAt` do evento e a entrega — **é a métrica que verifica o SLA de 10 s** (`[09:02] Marcos`) |
| `webhook_delivery_total` | counter | `result` (`success`/`failure`), `endpoint_id` | Taxa de sucesso por destino |
| `webhook_delivery_duration_seconds` | histogram | `endpoint_id` | Tempo de resposta do cliente; alimenta o diagnóstico de timeout |
| `webhook_delivery_attempts_total` | counter | `attempt` (1–5) | Distribuição das tentativas; concentração em 4–5 antecipa dead letter |
| `webhook_dead_letter_total` | counter | `endpoint_id` | Falhas definitivas — na ausência de alerta ativo ao cliente, é o principal sinal de degradação |
| `webhook_worker_loop_duration_seconds` | histogram | — | Duração do ciclo; se ultrapassar o intervalo de 2 s, o worker está atrasado |

### 8.2 Logs

Pino já existente (`src/shared/logger/index.ts`), sem nada novo (`[09:29] Bruno`). Eventos estruturados,
seguindo o padrão de nome do projeto (`http_request` em `request-logger.middleware.ts:23`):

| Evento | Nível | Campos |
| --- | --- | --- |
| `webhook_event_enqueued` | info | `eventId`, `outboxId`, `endpointId`, `orderId`, `toStatus`, `requestId` |
| `webhook_delivery_attempt` | info | `eventId`, `endpointId`, `attempt`, `responseStatus`, `durationMs` |
| `webhook_delivery_failed` | warn | `eventId`, `endpointId`, `attempt`, `errorCode`, `nextAttemptAt` |
| `webhook_dead_lettered` | error | `eventId`, `endpointId`, `attempts`, `failureReason` |
| `webhook_dead_letter_replayed` | warn | `deadLetterId`, `newOutboxId`, `replayedById` — **exigência de auditoria** (`[09:36] Sofia`) |
| `webhook_secret_rotated` | warn | `endpointId`, `previousSecretExpiresAt` — **nunca o valor da secret** |
| `webhook_worker_started` / `webhook_worker_stopped` | info | `pid`, `pollIntervalMs` |

> **Requisito de segurança derivado.** A lista `redact` do logger (`src/shared/logger/index.ts`, linhas
> 4–11) cobre `req.headers.authorization`, `req.headers.cookie`, `*.password`, `*.passwordHash`,
> `*.token` e `*.accessToken` — **não** cobre `*.secret`. Ela **precisa** ganhar `*.secret`,
> `*.previousSecret` e `*.signature`, sob pena de a decisão de
> [ADR-003](./adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md) ser anulada pelos nossos próprios
> logs. Esta é a única alteração que a feature impõe a um arquivo compartilhado fora do fluxo de pedidos.

### 8.3 Tracing e correlação

O projeto **não tem tracing distribuído** — não há OpenTelemetry nem equivalente no `package.json`, e a
reunião não tratou do assunto. Introduzi-lo contrariaria
[ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes.md). A correlação é obtida com o que existe:

- **`X-Request-Id`** já é gerado ou propagado por `src/middlewares/request-logger.middleware.ts:6` e
  devolvido no header da resposta. A requisição que muda o status **persiste esse `requestId` na linha da
  outbox**, e o worker o reemite em todo log da entrega. Isso costura, num único identificador, a
  requisição do operador → o evento → as até 5 tentativas → a dead letter → o replay.
- **`eventId`** correlaciona nosso lado ao lado do cliente: é o mesmo valor que viaja em `X-Event-Id`, o
  que permite investigar uma reclamação de duplicidade a partir do identificador que o cliente informa.
- **`outboxId`** liga cada linha de `webhook_deliveries` à sua tentativa.

Quando a plataforma adotar tracing distribuído, o ponto de propagação natural é a mesma coluna que hoje
carrega o `requestId`. Registrado como lacuna consciente na §3.

## 9. Dependências e compatibilidade

| Dependência | Situação |
| --- | --- |
| **Nenhuma biblioteca nova** | HMAC-SHA256 vem do módulo `node:crypto` (Node ≥ 20, `package.json`); o cliente HTTP vem do `fetch` nativo. Zod, Prisma, Express, Pino e uuid já estão no projeto. |
| **Migração Prisma** | Aditiva: 4 tabelas novas, nenhuma alteração em tabela existente. Aplicável com `npm run db:migrate`. |
| **Variáveis de ambiente** | `envSchema` (`src/config/env.ts:3`) ganha os parâmetros do worker (intervalo de polling, tamanho do lote, timeout). O worker reutiliza `DATABASE_URL` (`[09:30] Bruno`). |
| **Novo artefato de deploy** | `src/worker.ts` + script `npm run worker`, publicado e monitorado à parte da API (`[09:11] Larissa`). |
| **Compatibilidade retroativa** | **Total.** Nenhum contrato existente muda. `PATCH /orders/:id/status` mantém request e response; o único efeito observável novo é a gravação do evento — e apenas quando há endpoint assinante ([ADR-008](./adrs/ADR-008-filtragem-de-eventos-na-insercao.md)). |
| **Compatibilidade de dados** | Pedidos anteriores ao deploy não geram eventos retroativos: só transições posteriores entram na outbox. |
| **Testes existentes** | `tests/orders.test.ts` exercita `changeStatus` e deve continuar passando sem alteração — com zero endpoint cadastrado, nenhum evento é gravado. É a verificação prática da compatibilidade retroativa. |

## 10. Integração com o sistema existente

Todos os caminhos abaixo foram **verificados no repositório**.

### 10.1 `src/modules/orders/order.service.ts:126` — `changeStatus()`

**A única alteração de comportamento em código existente.** O método roda inteiro dentro de
`this.prisma.$transaction` (linha 131), onde já executa `tx.order.update` (158) e
`tx.orderStatusHistory.create` (159).

**Extensão:** uma chamada a `publishWebhookEvent(tx, order, from, to)` logo após o
`orderStatusHistory.create`, recebendo o mesmo `tx`. Sem novo parâmetro no construtor do `OrderService`,
sem `import` do repositório de webhooks — só a função, conforme `[09:41] Diego`. Se ela lançar, a
transação inteira reverte, que é o comportamento desejado (`[09:40] Bruno`).

### 10.2 `src/modules/orders/order.status.ts` — máquina de estados

**Não é alterada; é consultada.** O mapa `transitions` (linha 3) define o universo de transições que
podem gerar evento: `PENDING→PAID|CANCELLED`, `PAID→PROCESSING|CANCELLED`,
`PROCESSING→SHIPPED|CANCELLED`, `SHIPPED→DELIVERED`, com `DELIVERED` e `CANCELLED` terminais.

**Extensão:** o campo `subscribedStatuses` é validado contra o enum `OrderStatus`
(`prisma/schema.prisma:16`), e o filtro de [ADR-008](./adrs/ADR-008-filtragem-de-eventos-na-insercao.md)
compara `toStatus` com essa lista. `canTransition` (linha 12) continua sendo o guarda anterior: um evento
só nasce de uma transição que a máquina já aceitou.

### 10.3 `src/shared/errors/app-error.ts:3` e `src/shared/errors/http-errors.ts`

**Não são alterados; são estendidos.** As classes `WEBHOOK_*` da §6 herdam de `BadRequestError` (linha 3),
`NotFoundError` (27), `ConflictError` (33) e `UnprocessableEntityError` (39), replicando o padrão de
`InvalidStatusTransitionError` (45) e `InsufficientStockError` (55) — subclasse específica com código em
SCREAMING_SNAKE. As novas classes são reexportadas por `src/shared/errors/index.ts`.

### 10.4 `src/middlewares/error.middleware.ts:14` — handler central

**Zero alteração.** Herdando de `AppError`, os erros do módulo caem no ramo da linha 15 e são
serializados como `{ error: { code, message, details? } }` (linha 16). Os `ZodError` dos schemas do
módulo caem no ramo da linha 26. Confirmação de Bruno: "Vai pegar nossos erros sem precisar mudar nada"
(`[09:29] Bruno`).

### 10.5 `src/middlewares/auth.middleware.ts` — autenticação e papel

**Zero alteração.** `authenticate` (linha 27) protege todos os endpoints da §5. `requireRole` (linha 49)
protege o replay com `'ADMIN'` — hoje esse guard tem **um único uso no projeto**, em
`src/modules/users/user.routes.ts:15`, que serve de precedente exato. Note que `req.user` (linha 42) traz
o usuário **operador**, não o cliente: é por isso que `customerId` vai no corpo da requisição
(`[09:32] Bruno`, `[09:32] Larissa`), e é `req.user.id` que preenche `replayedById` na auditoria do
replay.

### 10.6 `src/routes/index.ts:21` e `src/app.ts:26` — registro e injeção

**Extensão aditiva.** Em `src/routes/index.ts`, o type `Controllers` (linha 13) ganha `webhooks` e
`buildApiRouter` ganha `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` mais o router
administrativo, ao lado das cinco linhas já existentes (24–28). Em `src/app.ts`, `buildControllers`
(linha 26) ganha o encadeamento `WebhookRepository → WebhookService → WebhookController`, idêntico ao de
orders (linhas 42–44).

### 10.7 `src/shared/logger/index.ts` — logger

**Uma alteração pontual e necessária:** acrescentar `*.secret`, `*.previousSecret` e `*.signature` ao
array `redactPaths` (linhas 4–11). Sem isso, a secret pode vazar nos nossos logs, anulando
[ADR-003](./adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md). É a **única** modificação que a feature
impõe a um arquivo compartilhado fora do fluxo de pedidos.

### 10.8 `src/middlewares/request-logger.middleware.ts:6` — correlação

**Zero alteração; é consumido.** O `requestId` (header de entrada ou UUID gerado) passa a ser persistido
na linha da outbox e reemitido pelo worker, costurando requisição → evento → tentativas → dead letter →
replay (§8.3).

### 10.9 `src/config/database.ts:4` e `src/server.ts` — processo do worker

`createPrismaClient()` (linha 4) é reaproveitado pelo worker para abrir **sua própria** instância — "
PrismaClient é por processo" (`[09:30] Bruno`). `src/server.ts` é o **modelo** da nova entry-point,
inclusive no encerramento gracioso com SIGINT/SIGTERM e `prisma.$disconnect()` (linhas 13–21). Larissa
descreveu exatamente isso: "Tipo o que a gente já tem em src/server.ts, criar um src/worker.ts e um
script npm run worker" (`[09:11] Larissa`).

### 10.10 `prisma/schema.prisma` — modelagem

**Extensão aditiva.** As 4 tabelas da §4.1 seguem as convenções verificadas: UUID em `@db.Char(36)`
(linhas 26, 75, 117), `@@map` snake_case (37, 96, 130), `Int` para centavos (61, 81).
`OrderStatusHistory` (linha 116) é o precedente direto: tabela auxiliar escrita na mesma transação, com
`fromStatus`/`toStatus` e índices por relação e por data (128–129). FKs novas apontam para `customers`
(linha 40) e `users` (25).

### 10.11 `src/shared/http/response.ts:22` e `tests/`

`paginated()` (linha 22) formata as listagens das §5.2 e §5.6, como já faz em
`order.service.ts:41`. Os testes reusam `tests/helpers/factories.ts` — `getTestApp` (10),
`bootstrapAuthenticatedUser` (76), `createTestCustomer` (42), `createTestProduct` (62) — sobre Vitest +
Supertest, o mesmo arranjo de `tests/orders.test.ts`.

## 11. Critérios de aceite técnicos

| ID | Critério | Como verificar |
| --- | --- | --- |
| **CAT-01** | O evento é gravado na mesma transação; se a gravação falhar, `order.status` e `order_status_history` permanecem inalterados. | Teste de integração forçando erro em `publishWebhookEvent` e conferindo o estado do pedido |
| **CAT-02** | Transição sem endpoint assinante **não** grava linha na outbox. | Teste com endpoint inativo e com `subscribedStatuses` que não inclui o `toStatus` |
| **CAT-03** | `tests/orders.test.ts` passa sem alteração. | `npm test -- tests/orders.test.ts` |
| **CAT-04** | O payload gravado é snapshot: alterar o pedido após a transição não muda o conteúdo entregue. | Teste que muda o pedido entre a inserção e o processamento |
| **CAT-05** | Payload acima de 64 KB dispara `WEBHOOK_PAYLOAD_TOO_LARGE` e aborta a transação. | Teste unitário do limite |
| **CAT-06** | `X-Signature` verifica com a secret do endpoint; um byte alterado no corpo invalida a verificação. | Teste unitário da assinatura |
| **CAT-07** | A secret não aparece em nenhuma resposta além da criação (§5.1) e da rotação (§5.5), nem em nenhum log. | Teste de contrato + inspeção do `redact` |
| **CAT-08** | Falhas geram `nextAttemptAt` em 1m/5m/30m/2h/12h; a 5ª falha grava dead letter e não reagenda. | Teste do agendador com relógio controlado |
| **CAT-09** | Destino que não responde em 10 s é tratado como falha. | Teste contra servidor com atraso deliberado |
| **CAT-10** | `X-Event-Id` é idêntico em todas as tentativas e preservado no replay. | Teste de integração ao longo do ciclo |
| **CAT-11** | Replay exige `ADMIN` (403 caso contrário) e grava `replayedById`. | Teste de API com token `OPERATOR` e com `ADMIN` |
| **CAT-12** | URL `http` é recusada com `WEBHOOK_INVALID_URL`. | Teste de API no cadastro e na alteração |
| **CAT-13** | Rotação mantém a secret anterior por 24 h e a limpa depois. | Teste com relógio controlado |
| **CAT-14** | Reiniciar a API não interrompe entregas pendentes. | Validação manual: derrubar a API com o worker rodando |
| **CAT-15** | Todos os erros do módulo respondem `{ error: { code, message } }` com prefixo `WEBHOOK_`. | Teste de contrato varrendo a matriz da §6 |
| **CAT-16** | `webhook_delivery_lag_seconds` fica abaixo de 10 s no caminho feliz. | Medição em ambiente de validação |

## 12. Riscos e mitigação

Os riscos de produto estão no [PRD §10](./PRD.md) e os arquiteturais no [RFC §6](./RFC.md). Aqui, apenas
os **de implementação**.

| ID | Risco | Mitigação |
| --- | --- | --- |
| **FDD-R-01** | **Esquecer de estender o `redact` do logger**, vazando a secret nos nossos logs e anulando o [ADR-003](./adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md). O risco é alto porque a alteração é em arquivo alheio ao módulo, fácil de passar despercebida. | CAT-07 como teste bloqueante e item explícito na revisão de segurança de Sofia (`[09:46] Sofia`) |
| **FDD-R-02** | **A consulta de endpoints assinantes dentro da transação** ([ADR-008](./adrs/ADR-008-filtragem-de-eventos-na-insercao.md)) aumenta o tempo da transação mais sensível do OMS — exatamente o peso que motivou recusar o síncrono (`[09:04] Bruno`). | Índice `(customerId, active)` em `webhook_endpoints`; consulta única por transição, nunca por endpoint |
| **FDD-R-03** | **Worker morto sem ninguém perceber.** Não há alerta ativo nesta fase, e a fila cresce silenciosamente. | `webhook_outbox_pending_total` e `webhook_worker_loop_duration_seconds` com alarme de operação |
| **FDD-R-04** | **Eventos travados em `PROCESSING`** após queda abrupta do worker, nunca retomados. | Recuperação por limiar de tempo (§7), coberta por teste |
| **FDD-R-05** | **Subir mais de uma instância do worker por engano de deploy**, quebrando a ordenação por pedido **silenciosamente** — nada no sistema impede. | Documentar a restrição no artefato de deploy; `[09:13] Larissa` a registra como limitação conhecida |
| **FDD-R-06** | **Assinar o corpo já reserializado**, gerando assinatura que não corresponde ao byte transmitido — causa clássica de falha de verificação no cliente. | Assinar exatamente o buffer enviado; CAT-06 |
| **FDD-R-07** | **Crescimento das 4 tabelas sem política de retenção** (`[09:08] Diego`, fora de escopo), degradando a leitura de pendentes. | Índice `(status, nextAttemptAt)` limita o custo à fatia pendente; monitorar volume e reabrir o tema com dados |
| **FDD-R-08** | **Ambiguidade não resolvida na reunião:** o arquivo de processamento ficou entre `webhook.worker.ts` e `webhook.processor.ts` (`[09:28] Bruno`), sem escolha registrada. | Decidir na sessão de revisão do design (`[09:50] Larissa`); não é bloqueante |
