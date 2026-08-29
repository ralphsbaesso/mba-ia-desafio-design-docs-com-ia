# 02 — Requisitos funcionais e não funcionais EXPLÍCITOS

Recorte: comportamento observável **dito na reunião**. Nada inferido. Itens descartados
(`03`) e adiados (`04`) não aparecem aqui.

## Requisitos funcionais (candidatos a `PRD-FR-NN`)

| # | Requisito | Localização |
| --- | --- | --- |
| RF-01 | O cliente pode **cadastrar um webhook** via `POST`, informando `url` e a lista de status que quer receber. A **secret é gerada pela plataforma e devolvida na criação**. | `[09:31] Marcos` |
| RF-02 | O `customer_id` do webhook é informado **no body ou no path**, não extraído do JWT. O endpoint é autenticado normalmente. | `[09:32] Larissa` |
| RF-03 | O cliente pode **editar** um webhook (`PATCH`). | `[09:33] Bruno` |
| RF-04 | O cliente pode **remover** um webhook (`DELETE`). | `[09:33] Bruno` |
| RF-05 | O cliente pode **listar** os webhooks de um customer (`GET`). | `[09:33] Bruno` |
| RF-06 | Cada endpoint de webhook define **quais status quer ouvir** (ex.: só `SHIPPED` e `DELIVERED`); a filtragem ocorre **na inserção da outbox**. | `[09:33] Marcos`; `[09:34] Bruno` |
| RF-07 | A configuração de webhook armazena **url + secret + customer_id + estado ativo**. | `[09:21] Bruno`; confirmado por `[09:21] Sofia` ("Sim") |
| RF-08 | O cliente pode **rotacionar a secret** pela API; a secret antiga continua válida por **24h** em paralelo e depois é invalidada. | `[09:21] Sofia` |
| RF-09 | O cliente pode **consultar o histórico de entregas** de um webhook — `GET /webhooks/:id/deliveries` — com sucesso/falha, payload, response e tempo de resposta (ex.: últimos 100 envios). | `[09:34] Marcos` |
| RF-10 | Um admin pode **reprocessar manualmente** um evento da DLQ via `POST /admin/webhooks/dead-letter/:id/replay`, recolocando-o na outbox como pendente. | `[09:18] Diego`; `[09:35] Diego` |
| RF-11 | O endpoint de replay exige **role `ADMIN`** e **registra quem executou o replay** para auditoria. | `[09:36] Sofia`; `[09:36] Larissa` |
| RF-12 | Toda mudança de status de pedido **gera um evento na outbox dentro da mesma transação**; falha na inserção causa rollback da mudança de status. | `[09:40] Bruno`; `[09:41] Diego` |
| RF-13 | O evento é entregue por **HTTP POST** ao endpoint do cliente com payload JSON contendo `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos da order (ex.: `total_cents`). Sem `items`. | `[09:43] Diego` |
| RF-14 | A requisição carrega os headers **`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`** e `Content-Type: application/json`. | `[09:44] Diego`; `[09:44] Sofia` |
| RF-15 | Entregas que falham são **retentadas até 5 vezes** com backoff 1m/5m/30m/2h/12h; esgotadas as tentativas o evento vai para a **DLQ**. | `[09:17] Diego`; `[09:17] Larissa` |
| RF-16 | O cadastro **recusa URLs não-`https`** com erro de validação. | `[09:23] Sofia` |
| RF-17 | Eventos com payload **acima de 64KB não são enviados** — a plataforma erra em vez de truncar. | `[09:23] Sofia` (erra, não trunca); `[09:24] Diego` (64KB); `[09:24] Larissa` (aceite) |
| RF-18 | O CRUD de configuração de webhook aceita **qualquer role autenticada**. | `[09:37] Sofia` |

## Requisitos não funcionais

| # | Requisito | Localização |
| --- | --- | --- |
| RNF-01 | **Latência de notificação < 10s** — é o que os clientes consideram "tempo real". | `[09:02] Marcos` |
| RNF-02 | O ciclo de polling do worker é de **2 segundos**, o que coloca o pior caso de latência de leitura em 2s. | `[09:09] Diego`; `[09:10] Larissa` |
| RNF-03 | A mudança de status **não pode ser bloqueada** por cliente lento ou indisponível — nada de HTTP call dentro da transação. | `[09:04] Bruno`; `[09:04] Bruno` (sobre rollback) |
| RNF-04 | **Timeout de 10s** por chamada HTTP ao cliente. | `[09:42] Diego` |
| RNF-05 | **Limite de 64KB** de payload por evento. | `[09:24] Diego`; `[09:24] Larissa` |
| RNF-06 | **Entrega at-least-once**: duplicidade é possível e o cliente dedupica por `X-Event-Id`. | `[09:24] Diego`; `[09:26] Larissa` |
| RNF-07 | **Ordering** garantida apenas **por `order_id`** e apenas enquanto houver **um único worker**; não há ordering global. | `[09:12] Diego`; `[09:13] Larissa` |
| RNF-08 | **TLS obrigatório** no endpoint de destino (`https`). | `[09:23] Sofia` |
| RNF-09 | **Secret única por endpoint** e rotacionável com grace period de 24h. | `[09:21] Sofia` |
| RNF-10 | **Integridade e autenticidade** do payload via HMAC-SHA256. | `[09:19] Sofia`; `[09:20] Sofia` |
| RNF-11 | O worker roda em **processo separado** da API para não morrer junto com um restart da API. | `[09:11] Diego` |
| RNF-12 | Nenhuma infraestrutura nova: **MySQL e Prisma já existentes**, sem Redis nem broker. | `[09:07] Diego` |
| RNF-13 | **Prazo de três sprints**, incluindo a revisão de segurança final. | `[09:46] Larissa`; `[09:47] Larissa` |
