# ADR-005 — Worker em processo separado, consumindo a outbox por polling

| | |
| --- | --- |
| **Status** | Aceita |
| **Data** | Reunião de definição da feature (`TRANSCRICAO.md`, quinta-feira, 09:00–09:53) |
| **Decisores** | Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Marcos (PM) |
| **RFC** | [RFC — Entrega de Notificações de Pedido por Webhook](../RFC.md), §3.2b |
| **Relacionadas** | [ADR-001](./ADR-001-outbox-no-mysql.md) (a fila que este worker consome) · [ADR-002](./ADR-002-retry-com-backoff-e-dlq.md) (a política que ele executa) |

## Contexto

Decidida a outbox ([ADR-001](./ADR-001-outbox-no-mysql.md)), restava definir **quem lê a tabela e como**:
"Próximo ponto: como o worker lê isso?" (`[09:08] Larissa`).

Duas restrições vindas do ambiente:

- **O requisito de latência é folgado.** Os clientes consideram "tempo real" qualquer coisa abaixo de 10
  segundos (`[09:02] Marcos`) — há orçamento de sobra para uma abordagem simples.
- **O MySQL não oferece notificação a processo externo.** Diferente do Postgres, não há `NOTIFY`/`LISTEN`
  nativo (`[09:09] Diego`).

E uma restrição de disponibilidade: o mecanismo de entrega não pode depender do ciclo de vida da API —
"se a API reinicia, perde o worker" (`[09:11] Diego`).

Há ainda a questão de ordem: se um pedido muda PAID → PROCESSING → SHIPPED em sequência rápida, o cliente
recebe na ordem certa? (`[09:12] Larissa`). A resposta depende diretamente do modelo de execução
escolhido.

## Decisão

**Um worker em processo separado da API, consumindo a outbox por polling a cada 2 segundos, em instância
única.**

- **Polling em loop, a cada 2 segundos**, buscando os eventos pendentes mais antigos, processando e
  marcando (`[09:09] Diego`). A latência mínima passa a ser de 2 segundos no pior caso, aceita
  explicitamente: "A latência mínima vai ser 2 segundos no pior caso. Aceitamos" (`[09:10] Larissa`), com
  o aval do PM: "2 segundos serve, perfeito" (`[09:10] Marcos`).
- **Processo separado, não a mesma instância da API.** "o worker tem que rodar como processo separado,
  não dentro da mesma instância da API" (`[09:11] Diego`). Concretamente, uma segunda entry-point ao lado
  da existente (`src/server.ts`) e um script dedicado no `package.json`, conforme proposto em
  `[09:11] Larissa`.
- **Mesmo banco, mesma stack, instância de client própria.** O worker é outro processo Node e por isso
  abre seu próprio `PrismaClient`, apontando para a mesma `DATABASE_URL`: "Separado. PrismaClient é por
  processo" (`[09:30] Bruno`), o que se apoia na fábrica já existente
  (`src/config/database.ts:4`, `createPrismaClient()`).
- **Instância única.** Com um único worker, o processamento segue a ordem de criação dos eventos e a
  ordenação **por pedido** fica preservada. Não há garantia de ordenação global, e a garantia por pedido
  vale enquanto não houver paralelismo — limitação registrada deliberadamente: "Não é garantia de
  ordering global, só por order_id e enquanto for single-worker" (`[09:13] Larissa`).
- **A lógica de processamento vive dentro do módulo de webhooks**, com a entry-point servindo apenas de
  bootstrap (`[09:28] Bruno`).

## Alternativas Consideradas

### A. Trigger de banco notificando o worker

Proposta por Bruno como caminho mais reativo: "Não dá pra usar trigger do banco pra ser mais reativo?"
(`[09:09] Bruno`).

**Rejeitada** por ausência do mecanismo. "MySQL não tem listener nativo tipo o NOTIFY/LISTEN do Postgres.
Trigger no banco a gente até tem, mas ela não notifica processo externo, ela só executa SQL. Pra avisar o
worker, a gente teria que improvisar algo tipo escrever em arquivo ou bater num endpoint, fica esquisito"
(`[09:09] Diego`). O substituto seria mais frágil e mais difícil de operar que o polling, sem ganho
proporcional: "Polling de 2 segundos atende o requisito de 'abaixo de 10 segundos' tranquilo".

### B. Worker dentro do mesmo processo da API

Rodar o loop de consumo como uma tarefa em segundo plano da própria aplicação Express.

**Rejeitada** por acoplamento de ciclo de vida — "se a API reinicia, perde o worker" (`[09:11] Diego`).
Traria ainda dois efeitos indesejados: a entrega concorreria por CPU e event loop com as requisições
HTTP, e cada réplica da API executaria seu próprio loop, criando o paralelismo — e a quebra de ordenação
— que a instância única existe justamente para evitar.

### C. Múltiplos workers em paralelo

Escalar horizontalmente o processamento desde o início.

**Rejeitada nesta fase** por quebrar a ordenação sem que exista demanda para tanto. "Se a gente escala pra
múltiplos workers em paralelo no futuro, perde a garantia" (`[09:12] Diego`); os caminhos de solução
foram nomeados — particionar por `order_id` ou usar lock pessimista — e adiados: "isso é problema do
futuro, não agora" (`[09:13] Diego`). Marcos confirmou que não há demanda: "Os clientes nunca pediram
garantia de ordering global" (`[09:14] Marcos`).

## Consequências

### Positivas

- **Isolamento total do fluxo de pedidos.** Nenhuma chamada externa acontece no caminho da requisição do
  usuário; um destino lento não afeta a API.
- **Independência operacional.** Deploy, reinício ou queda da API não interrompem a entrega de eventos
  pendentes, e vice-versa.
- **Simplicidade máxima de implementação.** Um loop com consulta e atualização, sem broker, sem
  coordenação, sem infraestrutura de notificação.
- **Ordenação por pedido de graça.** A instância única entrega a ordem correta por pedido sem nenhum
  mecanismo adicional.
- **Reuso da stack existente.** Mesmo banco, mesmo Prisma, mesma configuração — o worker é uma segunda
  entry-point do mesmo projeto, não um serviço novo.

### Negativas

- **Latência mínima de 2 segundos por construção.** Confortável dentro dos 10 s exigidos, mas é um piso
  que a arquitetura não consegue baixar sem mudar o mecanismo.
- **Consultas ociosas contínuas.** O polling consulta o banco a cada 2 segundos mesmo sem eventos
  pendentes — carga permanente, ainda que pequena, sobre o banco principal.
- **Ponto único de falha e teto de vazão.** Uma instância única significa que, se o worker cai, nada é
  entregue até ele voltar, e a vazão total é limitada ao que um processo consegue processar.
- **A ordenação é uma premissa operacional, não uma garantia do desenho.** Subir uma segunda instância —
  por engano de deploy ou por necessidade de escala — quebra a ordem por pedido **silenciosamente**. A
  limitação está documentada, mas nada no sistema a impõe.
- **Um artefato a mais para publicar e monitorar**, com seu próprio ciclo de vida, health check e
  alarmes.
