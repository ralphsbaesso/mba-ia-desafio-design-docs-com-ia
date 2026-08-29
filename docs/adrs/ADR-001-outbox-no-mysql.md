# ADR-001 — Padrão Outbox no MySQL existente

| | |
| --- | --- |
| **Status** | Aceita |
| **Data** | Reunião de definição da feature (`TRANSCRICAO.md`, quinta-feira, 09:00–09:53) |
| **Decisores** | Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos) |
| **RFC** | [RFC — Entrega de Notificações de Pedido por Webhook](../RFC.md), §3.2a |
| **Relacionadas** | [ADR-005](./ADR-005-worker-separado-em-polling.md) (quem consome a outbox) · [ADR-007](./ADR-007-snapshot-do-payload-na-insercao.md) (o que a outbox guarda) · [ADR-008](./ADR-008-filtragem-de-eventos-na-insercao.md) (o que chega a ser gravado) |

## Contexto

O OMS precisa notificar clientes B2B a cada mudança de status de pedido, e o sistema **não possui hoje
nenhum mecanismo de eventos, filas ou notificação externa**. A pergunta que abriu a discussão foi
exatamente essa: "a gente dispara isso sincronamente no service de orders quando o status muda, ou faz
algum tipo de fila/outbox?" (`[09:03] Larissa`).

Duas propriedades do sistema existente restringem a resposta:

1. **A mudança de status já é uma transação pesada.** A operação valida a transição, movimenta estoque,
   atualiza o pedido e grava o histórico de status — tudo dentro de uma transação
   (`src/modules/orders/order.service.ts:126`, cujo corpo roda inteiro em `prisma.$transaction`, linha
   131). Acrescentar uma chamada HTTP no meio disso faria "qualquer cliente lento travar mudança de
   status pra outros pedidos" (`[09:04] Bruno`).
2. **O time é pequeno** (`[09:07] Diego`), e cada componente novo de infraestrutura carrega custo
   operacional permanente.

Há ainda uma exigência de correção que não admite meio-termo: não pode existir o caso de o status mudar e
o evento não sair — "Não pode ter caso de status mudar e evento não sair" (`[09:40] Bruno`). Qualquer
mecanismo que registre o evento *fora* da transação do pedido abre uma janela em que os dois estados
divergem.

## Decisão

Adotamos o **padrão outbox sobre o MySQL já existente**.

Quando o status do pedido muda, **dentro da mesma transação SQL** que atualiza o pedido e grava o
histórico, uma linha com o evento é inserida numa tabela de outbox. Um processo separado lê essa tabela e
dispara as chamadas HTTP.

A propriedade que isso compra foi descrita por Diego: "se a transação principal commitou, o evento foi
registrado, e se ela deu rollback, o evento some junto. Não tem inconsistência possível"
(`[09:06] Diego`). A decisão foi fechada em `[09:08] Larissa`: "Tá decidido então: outbox em MySQL".

Detalhes de desenho fixados junto com a decisão:

- **Índices** no status do evento (pendente, processando, falhou, entregue) e na data de criação; o
  worker lê apenas os pendentes, em **batch pequeno**, e marca como entregue (`[09:08] Diego`).
- **Chave primária em UUID**, seguindo a convenção do resto do projeto — "UUID, segue o padrão do resto
  do projeto. Tudo é uuid" (`[09:51] Larissa`), convenção verificável em
  `prisma/schema.prisma` (`@id @default(uuid()) @db.Char(36)`, ex.: linhas 26, 75 e 117).
- **A escrita é feita por uma função que recebe a transação em curso** — `publishWebhookEvent(tx, order,
  fromStatus, toStatus)` — em vez de injetar o repositório de webhooks dentro do serviço de pedidos:
  "função pura recebendo o tx. Não precisa injetar repository inteiro" (`[09:41] Diego`).

A tabela `order_status_history` (`prisma/schema.prisma:116`) é o precedente direto de modelagem: já é uma
tabela auxiliar escrita na mesma transação, com `fromStatus`/`toStatus` e índices por pedido e por data.

## Alternativas Consideradas

### A. Disparo síncrono da chamada HTTP dentro da transação

Chamar o endpoint do cliente no próprio `changeStatus`, sem tabela intermediária.

**Rejeitada.** Acoplaria a disponibilidade do fluxo de pedidos à de terceiros. Bruno levantou os dois
problemas: um cliente lento travaria mudanças de status de outros pedidos, e com o cliente fora do ar não
haveria saída aceitável — "o que a gente faz, dá rollback na mudança de status? Não dá"
(`[09:04] Bruno`). Diego encerrou o ponto: "Síncrono está fora de questão" (`[09:06] Diego`).

### B. Redis Streams ou infraestrutura de mensageria dedicada

Publicar o evento num broker externo em vez de uma tabela.

**Rejeitada.** Levantada por Larissa como alternativa natural (`[09:07] Larissa`), foi descartada pelo
custo operacional: "a gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no
MySQL existente resolve" (`[09:07] Diego`). Há também um problema de correção que o broker não resolve
sozinho: publicar no broker não é transacional com o commit do MySQL, o que reintroduziria exatamente a
janela de divergência que a outbox elimina.

## Consequências

### Positivas

- **Consistência garantida por construção.** Evento e mudança de status commitam ou revertem juntos; não
  existe estado intermediário observável.
- **Custo de infraestrutura zero.** Nenhum componente novo a provisionar, monitorar ou pagar — o banco já
  existe e já está no caminho crítico da operação.
- **Durabilidade herdada.** Os eventos ganham de graça as garantias de persistência, backup e restore do
  banco principal.
- **Depuração trivial.** O estado da fila é inspecionável com uma consulta SQL comum, sem ferramental
  novo.
- **Coerência com a codebase.** A escrita é mais um insert numa transação que já faz dois, seguindo o
  padrão de `order_status_history`.

### Negativas

- **A atomicidade tem duas faces.** Se a inserção do evento falhar, a mudança de status **também** falha.
  Um defeito no mecanismo de notificação se propaga para o fluxo mais crítico do OMS. Este é o trade-off
  central da decisão, e foi aceito explicitamente (`[09:40] Bruno`, `[09:41] Diego`): a alternativa —
  perder eventos silenciosamente — foi considerada pior.
- **A tabela cresce indefinidamente.** Sem política de retenção, o volume acumulado degrada a leitura dos
  pendentes. Mitigado no curto prazo pelos índices e pela leitura em lotes pequenos; o arquivamento foi
  reconhecido como necessário e colocado **fora do escopo** desta feature (`[09:08] Diego`).
- **Carga adicional no banco principal.** A outbox concorre por I/O com a operação transacional do OMS,
  tanto na escrita quanto no polling do worker.
- **Entrega não é imediata por construção.** O evento fica registrado, mas só sai quando o consumidor o
  processa — a latência resultante é tratada em [ADR-005](./ADR-005-worker-separado-em-polling.md).
- **Mecanismo próprio a manter.** Ao não usar um broker pronto, a plataforma assume a manutenção da
  lógica de fila, retentativa e estados do evento.
