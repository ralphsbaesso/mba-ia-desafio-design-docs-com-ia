# ADR-008 — Filtragem de eventos por assinatura na inserção da outbox

| | |
| --- | --- |
| **Status** | Aceita |
| **Data** | Reunião de definição da feature (`TRANSCRICAO.md`, quinta-feira, 09:00–09:53) |
| **Decisores** | Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Marcos (PM) |
| **RFC** | [RFC — Entrega de Notificações de Pedido por Webhook](../RFC.md), §3.3 |
| **Relacionadas** | [ADR-001](./ADR-001-outbox-no-mysql.md) (a inserção que este filtro condiciona) · [ADR-005](./ADR-005-worker-separado-em-polling.md) (o consumo que ele alivia) |

## Contexto

Cada endpoint de webhook assina apenas as mudanças de status que interessam ao cliente. Marcos descreveu
o requisito: "Filtro de eventos é uma lista dos status que o webhook quer ouvir. Tipo 'só quero saber
quando vira SHIPPED e DELIVERED'" (`[09:33] Marcos`).

Isso cria uma escolha de arquitetura que Diego formulou diretamente: "Filtra na inserção do outbox ou na
hora de mandar?" (`[09:34] Diego`).

O contexto quantitativo importa: a máquina de estados do OMS
(`src/modules/orders/order.status.ts`) admite transições a partir de todos os estados não terminais —
PENDING, PAID, PROCESSING e SHIPPED — de modo que a maioria dos pedidos atravessa várias transições ao
longo do ciclo de vida. Se cada uma delas gerar linha na outbox independentemente de haver interessado,
a tabela cresce com trabalho que será descartado depois — e ela já cresce sem política de retenção
([ADR-001](./ADR-001-outbox-no-mysql.md)).

## Decisão

**A filtragem acontece na inserção: se nenhum endpoint ativo do cliente assina aquela transição, o evento
não é gravado na outbox.**

Bruno decidiu: "Na inserção. Se nenhum webhook do customer quer aquele status, nem insere. Economiza
linha na tabela" (`[09:34] Bruno`). Diego concordou (`[09:34] Diego`).

Consequência direta: a avaliação das assinaturas ocorre **dentro da transação de mudança de status**,
junto com a inserção do evento ([ADR-001](./ADR-001-outbox-no-mysql.md)) — a decisão de gravar ou não
depende de consultar as configurações de webhook do cliente naquele instante.

## Alternativas Consideradas

### A. Filtrar no momento do envio

Gravar toda mudança de status na outbox e deixar o worker decidir, na hora de entregar, quais endpoints
devem receber cada evento.

**Rejeitada** pelo custo de armazenamento e processamento sem contrapartida — "Economiza linha na tabela"
(`[09:34] Bruno`). Eventos que ninguém assina consumiriam espaço, seriam lidos pelo polling e depois
descartados, agravando o crescimento da outbox e o custo de cada ciclo do worker.

A alternativa tem um mérito real que foi preterido: **a assinatura passa a ser avaliada no momento da
transição, não no da entrega**. Um cliente que ative um endpoint logo após uma mudança de status não
recebe aquele evento — com a filtragem no envio, receberia. Isso não foi discutido na reunião; a decisão
foi tomada pelo critério de economia.

## Consequências

### Positivas

- **A outbox só contém trabalho útil.** Nenhuma linha é gravada para ser descartada depois.
- **Cada ciclo do worker é mais barato.** O polling lê apenas eventos que efetivamente têm destino.
- **Menor pressão sobre o crescimento da tabela**, que não tem política de retenção nesta fase.
- **Semântica clara para o histórico de entregas.** Todo evento na outbox tem ao menos um destinatário,
  o que torna a consulta de entregas do cliente direta.

### Negativas

- **Trabalho a mais dentro da transação crítica.** Avaliar as assinaturas exige uma consulta às
  configurações de webhook dentro da transação de mudança de status — a transação mais sensível do OMS,
  cujo peso foi exatamente o argumento contra o disparo síncrono (`[09:04] Bruno`). A consulta é local ao
  banco, sem chamada externa, mas não é gratuita.
- **A assinatura é congelada no instante da transição.** Ativar um endpoint ou incluir um status novo na
  assinatura **não** alcança eventos anteriores — eles nunca existiram. Não há como reprocessar
  retroativamente, e a dead letter não ajuda, porque não houve evento.
- **Nenhum rastro das transições não assinadas.** A outbox deixa de ser um registro completo do que
  aconteceu, passando a ser um registro do que precisava ser entregue. Para auditoria da transição em si,
  a fonte continua sendo `order_status_history` (`prisma/schema.prisma:116`).
- **Acoplamento adicional entre os módulos.** O caminho de mudança de status passa a depender da
  configuração de webhooks para decidir o que gravar, e não apenas para gravar.
