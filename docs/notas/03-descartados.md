# 03 — O que foi EXPLICITAMENTE DESCARTADO

Recorte: alternativas colocadas na mesa e **rejeitadas na própria reunião**, com o motivo
declarado. Alimenta "Fora de escopo" (PRD) e "Alternativas consideradas" (RFC/ADRs).

Nada aqui pode aparecer como requisito em nenhum documento.

| # | Descartado | Motivo declarado | Localização |
| --- | --- | --- | --- |
| DESC-01 | **Disparo síncrono do webhook dentro do service de orders** | A transação de mudança de status já é pesada (update em `orders`, insert em `order_status_history`, decremento de estoque); um HTTP call no meio faz cliente lento travar mudança de status de outros pedidos. E se o cliente estiver fora do ar não há o que fazer — dar rollback na mudança de status não é opção. | `[09:04] Bruno` (dois trechos); `[09:06] Diego` ("Síncrono está fora de questão") |
| DESC-02 | **Redis Streams / infra de fila dedicada** | Exigiria subir mais infraestrutura; o time é pequeno e subir Redis Cluster para isso é overengineering. Outbox no MySQL existente resolve. | `[09:07] Larissa` (levanta a alternativa); `[09:07] Diego` (descarta) |
| DESC-03 | **Trigger de banco para notificar o worker** (em vez de polling) | MySQL não tem listener nativo como o `NOTIFY`/`LISTEN` do Postgres; a trigger só executa SQL e não notifica processo externo. Avisar o worker exigiria improviso (escrever em arquivo, bater num endpoint). Polling de 2s já atende o requisito de <10s. | `[09:09] Bruno` (propõe); `[09:09] Diego` (descarta) |
| DESC-04 | **3 tentativas de retry** (alternativa a 5) | Pouco: cliente com indisponibilidade matinal seria retentado três vezes em ~30 minutos e o evento morreria. Já houve cliente com 2h de indisponibilidade em manutenção planejada. | `[09:16] Bruno` (propõe); `[09:16] Diego` (descarta) |
| DESC-05 | **Retry indefinido com backoff** | Deixa o evento pendurado para sempre se o cliente sumiu. | `[09:15] Diego` |
| DESC-06 | **DLQ como flag `failed` na própria tabela de outbox** | Tabela `webhook_dead_letter` separada mantém a leitura da outbox principal mais limpa e serve de evidência para debug e reprocessamento. | `[09:17] Larissa` (coloca as duas opções); `[09:18] Diego` (escolhe a tabela separada) |
| DESC-07 | **Secret global da plataforma** (uma só para todos os endpoints) | "Se vaza uma, vaza tudo." | `[09:21] Sofia` |
| DESC-08 | **Entrega exactly-once** | Exigiria coordenação dos dois lados e ficaria muito mais complexo; at-least-once com `event_id` resolve 99% dos casos e é o padrão de mercado (Stripe, GitHub). | `[09:25] Diego` |
| DESC-09 | **Truncar payload acima do limite de tamanho** | Se o evento chegou a esse tamanho, tem algo errado — erra em vez de truncar. | `[09:23] Sofia`; aceite em `[09:24] Larissa` |
| DESC-10 | **Notificação por e-mail ao cliente quando o webhook falha** (ex.: 3 falhas seguidas) | Fora de escopo desta fase; eventualmente na próxima, depois de medir o impacto. | `[09:37] Marcos` (propõe); `[09:37] Larissa` (descarta desta fase) |
| DESC-11 | **Dashboard/painel visual para o cliente ver seus webhooks** | "Agora não. Só endpoints." Painel é projeto separado do time de frontend. | `[09:39] Marcos` (propõe); `[09:40] Larissa` (descarta) |
| DESC-12 | **Webhooks inbound** (cliente enviando eventos para a plataforma) | Os clientes querem receber, não mandar — o escopo é outbound apenas. | `[09:02] Sofia` (pergunta); `[09:02] Marcos` (define outbound) |
| DESC-13 | **Compartilhar o mesmo `PrismaClient`/processo entre API e worker** | `PrismaClient` é por processo; o worker é outro processo Node e por isso abre instância própria. E o worker não pode viver na mesma instância da API — se a API reinicia, perde o worker. | `[09:11] Diego`; `[09:30] Bruno` |
| DESC-14 | **Injetar o repository de webhooks inteiro no `OrderService`** | Basta uma função pura recebendo o `tx` da transação atual — `publishWebhookEvent(tx, order, fromStatus, toStatus)`. | `[09:41] Bruno` (propõe a função); `[09:41] Diego` ("Não precisa injetar repository inteiro") |
| DESC-15 | **Guardar na outbox apenas o `order_id` e renderizar o payload na hora do envio** | Se o pedido mudar depois, o evento deixaria de refletir o estado do momento da mudança de status. Fica snapshot renderizado na inserção. | `[09:51] Bruno` (levanta); `[09:52] Larissa` (descarta); `[09:52] Diego` e `[09:52] Bruno` (ratificam) |
| DESC-16 | **`id` auto incremental na outbox** | UUID segue o padrão do resto do projeto — "tudo é uuid". | `[09:51] Diego` (pergunta); `[09:51] Larissa` (decide UUID) |
| DESC-17 | **Enviar `items` do pedido no payload** | Mantém o payload enxuto e não infla o evento; se o cliente quiser detalhe, bate no `GET /orders/:id`. | `[09:43] Diego`; `[09:44] Bruno` |
| DESC-18 | **Adicionar ferramenta/stack nova de logging** | Pino já está no projeto inteiro e o middleware de erro centralizado já trata `AppError`, Zod e Prisma — não precisa mudar nada. | `[09:29] Bruno` |
