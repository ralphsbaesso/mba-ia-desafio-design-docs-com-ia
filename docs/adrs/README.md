# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature **Sistema de Webhooks de
Notificação de Pedidos**.

Cada decisão arquitetural fechada é registrada em um arquivo individual, nomeado no formato
`ADR-NNN-titulo-em-kebab-case.md` (por exemplo `ADR-001-outbox-no-mysql.md`). Todos seguem o formato
**MADR**, com as seções Status, Contexto, Decisão, Alternativas Consideradas e Consequências — estas
últimas sempre separadas em positivas e negativas, com o trade-off explícito.

A proposta arquitetural que conecta as decisões está no [RFC](../RFC.md); o detalhamento de
implementação, no [FDD](../FDD.md).

## Índice

| ADR | Decisão | Origem na reunião |
| --- | --- | --- |
| [ADR-001](./ADR-001-outbox-no-mysql.md) | **Padrão outbox no MySQL existente** — o evento é gravado na mesma transação que muda o status do pedido | `[09:08] Larissa` |
| [ADR-002](./ADR-002-retry-com-backoff-e-dlq.md) | **Retentativa com backoff exponencial e DLQ** — 5 tentativas em 1m/5m/30m/2h/12h, depois dead letter com replay manual | `[09:17] Larissa` |
| [ADR-003](./ADR-003-hmac-sha256-com-secret-por-endpoint.md) | **HMAC-SHA256 com secret por endpoint** — assinatura do corpo, credencial exclusiva por destino, rotação com 24 h de convivência | `[09:22] Sofia` |
| [ADR-004](./ADR-004-entrega-at-least-once-com-x-event-id.md) | **Entrega at-least-once com `X-Event-Id`** — duplicidade é possível e o cliente deduplica pelo identificador do evento | `[09:26] Larissa` |
| [ADR-005](./ADR-005-worker-separado-em-polling.md) | **Worker em processo separado, em polling** — consumo da outbox a cada 2 segundos, fora da instância da API | `[09:11] Diego` |
| [ADR-006](./ADR-006-reuso-dos-padroes-existentes.md) | **Reuso dos padrões existentes** — nenhuma dependência ou convenção nova; é o ADR ancorado no código | `[09:30] Larissa` |
| [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) | **Snapshot do payload na inserção** — o evento guarda o estado do pedido no momento da transição | `[09:52] Larissa` |
| [ADR-008](./ADR-008-filtragem-de-eventos-na-insercao.md) | **Filtragem de eventos na inserção** — transição que nenhum endpoint ativo assina não gera evento | `[09:34] Bruno` |

As seis primeiras cobrem as decisões principais da reunião. ADR-007 e ADR-008 registram decisões
secundárias que têm contexto, alternativa real e trade-off próprios na transcrição.
