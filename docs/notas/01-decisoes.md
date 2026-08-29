# 01 — Decisões técnicas FECHADAS

Recorte: apenas decisões **fechadas** na reunião. Não entram ideias em discussão,
descartadas (→ `03-descartados.md`) ou adiadas (→ `04-adiados.md`).

As 6 decisões principais exigidas pelo enunciado estão marcadas com **[P1]**–**[P6]**.

| # | Decisão | Localização |
| --- | --- | --- |
| D-01 | **[P1]** Padrão **outbox no MySQL existente**: a mudança de status insere o evento numa tabela `webhook_outbox` dentro da mesma transação SQL que atualiza `orders` e `order_status_history`. | `[09:06] Diego`, ratificada em `[09:08] Larissa` ("Tá decidido então: outbox em MySQL") |
| D-02 | Disparo **síncrono** dentro do service de orders está **fora de questão**. | `[09:06] Diego` |
| D-03 | Outbox indexada por **status do evento** (pendente/processando/falhou/entregue) e por **created_at**; worker lê pendentes em **batch pequeno** e marca como entregue. | `[09:08] Diego` |
| D-04 | **[P5]** Worker consome por **polling em loop a cada 2 segundos**, buscando os eventos pendentes mais antigos. Latência mínima de 2s no pior caso é aceita explicitamente. | `[09:09] Diego`; aceite em `[09:10] Larissa` |
| D-05 | **[P5]** Worker roda como **processo separado** da API (nova entry-point `src/worker.ts` + script `npm run worker`), mesmo banco e mesma stack. | `[09:11] Diego` (processo separado), `[09:11] Larissa` (entry-point/script) |
| D-06 | Worker abre um **PrismaClient próprio** (instância nova), mesma `DATABASE_URL` — PrismaClient é por processo. | `[09:30] Bruno` |
| D-07 | **Single-worker** por ora; ordering **implícita por `order_id`** via `created_at` da outbox. Não há garantia de ordering global — registrada como limitação conhecida. | `[09:12] Diego`; `[09:13] Larissa` |
| D-08 | **[P2]** **Retry com backoff exponencial**, **5 tentativas**, progressão **1m / 5m / 30m / 2h / 12h** (~15h entre a primeira falha e a última tentativa). | `[09:17] Diego` (progressão); `[09:17] Larissa` ("Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h") |
| D-09 | **[P2]** Esgotadas as tentativas, o evento vai para uma **DLQ em tabela separada** (`webhook_dead_letter`) com payload, motivo da falha e timestamp. | `[09:18] Diego`; `[09:17] Larissa` (pergunta que fecha o ponto) |
| D-10 | **Reprocessamento manual da DLQ** via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente. | `[09:18] Diego`; `[09:35] Diego` (confirma o path) |
| D-11 | O endpoint de replay exige **role `ADMIN`**, reaproveitando o `requireRole` existente, e **loga quem fez o replay** para auditoria. | `[09:36] Sofia` (ADMIN + log de auditoria); `[09:36] Larissa` (reuso do `requireRole`) |
| D-12 | **[P3]** Assinatura **HMAC-SHA256 sobre o corpo do request**, enviada no header `X-Signature`. | `[09:20] Sofia` (HMAC + header); `[09:20] Sofia` (SHA-256); consolidado em `[09:22] Sofia` |
| D-13 | **[P3]** **Secret única por endpoint** de webhook (não global da plataforma). | `[09:21] Sofia` |
| D-14 | **[P3]** Secret **rotacionável por API**, com a secret antiga válida em paralelo por **grace period de 24h**. | `[09:21] Sofia`; consolidado em `[09:22] Sofia` |
| D-15 | **TLS obrigatório**: URL de webhook precisa ser `https`; `http` é recusado com erro de validação (validação de schema Zod, não decisão arquitetural). | `[09:23] Sofia` |
| D-16 | **[P4]** Garantia de entrega **at-least-once**; o cliente pode receber o mesmo evento mais de uma vez. | `[09:24] Diego`; ratificado em `[09:26] Larissa` |
| D-17 | **[P4]** Header **`X-Event-Id`** com UUID gerado na inserção do evento na outbox, para dedup do lado do cliente. | `[09:25] Diego`; `[09:26] Larissa` |
| D-18 | **[P6]** Módulo `src/modules/webhooks` seguindo o padrão da codebase (controller, service, repository, routes, schemas). | `[09:27] Bruno`; `[09:28] Diego` ("Faz") |
| D-19 | **[P6]** Lógica de processamento do worker num arquivo do módulo (`webhook.worker.ts` / `webhook.processor.ts`), com `src/worker.ts` só como entry. | `[09:28] Bruno`; `[09:28] Diego` |
| D-20 | **[P6]** Códigos de erro com **prefixo `WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) seguindo o padrão `AppError` existente. | `[09:28] Bruno`; `[09:29] Larissa` ("Prefixo WEBHOOK_ pra tudo do módulo") |
| D-21 | **[P6]** **Reuso máximo** do que já existe: `AppError`, Pino, error middleware centralizado, padrão de módulos, padrão de schemas Zod, padrão de códigos de erro. Nada novo de logging. | `[09:29] Bruno`; `[09:30] Larissa` ("Decisão: reuso máximo do que já existe") |
| D-22 | O evento é escrito na outbox **dentro da mesma transação** do `changeStatus`; se a inserção na outbox falhar, **rollback** de tudo. | `[09:40] Bruno`; `[09:41] Diego` ("Essencial") |
| D-23 | A escrita é feita por uma **função que recebe o `tx` da transação atual** — `publishWebhookEvent(tx, order, fromStatus, toStatus)` — em vez de injetar o repository inteiro no `OrderService`. | `[09:41] Bruno`; `[09:41] Diego` |
| D-24 | **Filtro de eventos por endpoint**: cada webhook assina uma lista de status; a filtragem acontece **na inserção da outbox** (se nenhum webhook do customer quer aquele status, nem insere). | `[09:33] Marcos` (filtro); `[09:34] Bruno` (na inserção); `[09:34] Diego` ("Concordo") |
| D-25 | `customer_id` **não vem do JWT**: é passado no body ou no path. O JWT atual é de usuário operador do nosso sistema. | `[09:32] Larissa` |
| D-26 | CRUD de configuração de webhook exige apenas **autenticação** (qualquer role autenticada) — por enquanto. | `[09:37] Sofia` |
| D-27 | **Timeout do HTTP call** do worker: **10 segundos**; estouro é tratado como falha e marcado para retry. | `[09:42] Diego` |
| D-28 | **Payload enxuto em JSON**: `event_id`, `event_type` (`"order.status_changed"`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order (ex.: `total_cents`). **Sem `items`** — detalhe fica no `GET /orders/:id`. | `[09:43] Diego`; `[09:44] Bruno` |
| D-29 | **Headers do request**: `X-Event-Id`, `X-Signature`, `X-Timestamp` (timestamp do envio, para o cliente detectar replay attack), `Content-Type: application/json` e `X-Webhook-Id` (id do endpoint cadastrado). | `[09:44] Diego`; `[09:44] Sofia` (`X-Webhook-Id`); `[09:45] Diego` (aceite) |
| D-30 | **`id` da outbox em UUID**, seguindo o padrão do resto do projeto. | `[09:51] Larissa` |
| D-31 | O evento guarda o **payload já renderizado (snapshot na inserção)**, não apenas `order_id` — para o evento refletir o estado do momento da mudança de status. | `[09:52] Larissa`; `[09:52] Diego`; `[09:52] Bruno` ("Decidido") |
| D-32 | **Escopo:** webhooks são **outbound apenas** (da plataforma para o cliente); o cliente não envia webhooks para nós. | `[09:02] Marcos`; `[09:03] Sofia` |
| D-33 | **Prazo: três sprints**, já incluindo a revisão de segurança da Sofia ao final. | `[09:46] Larissa`; `[09:47] Larissa` |
| D-34 | Sofia reserva **no mínimo 2 dias úteis** para revisão de segurança (HMAC e geração de secret) **antes do deploy**. | `[09:46] Sofia`; `[09:49] Sofia` |

## Cobertura das 6 decisões principais do enunciado

| Decisão principal | Coberta por | Timestamp âncora |
| --- | --- | --- |
| P1 — Outbox no MySQL | D-01 | `[09:08] Larissa` |
| P2 — Retry com backoff + DLQ | D-08, D-09 | `[09:17] Larissa` |
| P3 — HMAC-SHA256 com secret por endpoint | D-12, D-13, D-14 | `[09:22] Sofia` |
| P4 — At-least-once com `X-Event-Id` | D-16, D-17 | `[09:26] Larissa` |
| P5 — Worker separado em polling | D-04, D-05 | `[09:10] Larissa` / `[09:11] Diego` |
| P6 — Reuso dos padrões existentes | D-18, D-19, D-20, D-21 | `[09:30] Larissa` |
