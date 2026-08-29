# RFC — Entrega de Notificações de Pedido por Webhook

## Metadados

| | |
| --- | --- |
| **Autor** | Larissa — Tech Lead |
| **Status** | Aprovado — decisões ratificadas na reunião de definição (`[09:48] Larissa`) |
| **Data** | Quinta-feira, 09:00 — reunião de definição, ~55 min |
| **Revisores** | Larissa (Tech Lead) · Marcos (Product Manager) · Bruno (Eng. Pleno, Pedidos) · Diego (Eng. Sênior, Plataforma) · Sofia (Eng. de Segurança) |
| **Aprovação** | `[09:49] Diego` · `[09:49] Bruno` · `[09:49] Marcos` · `[09:49] Sofia`, condicionada à revisão de segurança pré-deploy |
| **Relacionados** | [PRD](./PRD.md) · [FDD (TDD)](./FDD.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md) |

> **Altura deste documento.** O RFC responde *como pretendemos resolver, e o que ainda está em aberto*.
> O *porquê* está no [PRD](./PRD.md); cada decisão fechada tem seu [ADR](./adrs/); contratos HTTP,
> payloads, esquema de tabelas e matriz de erros estão no [FDD](./FDD.md) e **não são repetidos aqui**.

---

## 1. TL;DR

Propomos entregar notificações de mudança de status de pedido por **webhook outbound**, usando o padrão
**outbox** sobre o MySQL que já temos: a mudança de status e o registro do evento acontecem na **mesma
transação**, e um **processo separado** consome os eventos pendentes **por polling**, entregando-os por
HTTP ao endpoint do cliente.

A entrega é **at-least-once**, com identificador único por evento para o cliente deduplicar; falhas são
**retentadas 5 vezes com espaçamento crescente** e, esgotadas as tentativas, o evento é preservado em uma
fila de mortos com reprocessamento manual por administrador. Cada notificação é **assinada com HMAC-SHA256
usando uma credencial exclusiva do endpoint**, rotacionável com 24 h de convivência.

A proposta **não introduz nenhuma infraestrutura nova** e reaproveita ao máximo os padrões já
estabelecidos na codebase. Esforço estimado: **3 sprints**, com a revisão de segurança inclusa
(`[09:47] Larissa`).

## 2. Contexto e problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — pediram formalmente para serem
notificados quando o status dos seus pedidos muda. Hoje eles fazem polling na API de pedidos, o que
torna a integração "lenta e cara" para eles (`[09:00] Marcos`), e a Atlas condicionou a permanência na
plataforma à entrega até o fim do trimestre.

O OMS **não tem hoje nenhum mecanismo de eventos, filas ou notificação externa**: não há legado de
mensageria a acomodar, e há liberdade para escolher o mecanismo mais barato que satisfaça o requisito. E
o requisito de latência é generoso — os clientes consideram "tempo real" **qualquer coisa abaixo de 10
segundos** (`[09:02] Marcos`), o que afasta a necessidade de uma solução *push* de baixa latência.

Duas restrições enquadram a decisão: **o fluxo de mudança de status já é pesado** (atualiza o pedido,
insere no histórico e movimenta estoque, tudo em uma transação — `[09:04] Bruno`) e **o time é pequeno**
(`[09:07] Diego`), de modo que cada peça nova de infraestrutura tem custo operacional permanente.

O problema central, portanto, não é *como notificar*, e sim: **como garantir que nenhuma mudança de
status deixe de gerar notificação, sem que a notificação possa atrapalhar a mudança de status.**

## 3. Proposta técnica

### 3.1 Visão geral

```
   PATCH status do pedido
            │
            ▼
   ┌───────────────────────────────────────────┐
   │  TRANSAÇÃO ÚNICA                          │
   │   • valida transição de estado            │
   │   • movimenta estoque                     │
   │   • atualiza o pedido                     │
   │   • grava o histórico de status           │
   │   • ► grava o evento na OUTBOX            │  ← tudo commita, ou nada commita
   └───────────────────────────────────────────┘
            │
            │  (assíncrono, outro processo)
            ▼
   ┌───────────────────────────────────────────┐
   │  WORKER — processo separado, polling 2 s  │
   │   lê pendentes mais antigos em batch      │
   └───────────────────────────────────────────┘
            │
            ▼  POST assinado (HMAC-SHA256), timeout 10 s
      ┌───────────┐   sucesso → entregue
      │  CLIENTE  │   falha   → reprograma (1m, 5m, 30m, 2h, 12h)
      └───────────┘   5ª falha → DEAD LETTER → replay manual (ADMIN)
```

### 3.2 Os cinco pilares da proposta

**a) Atomicidade pelo padrão outbox.** O evento é gravado numa tabela de outbox **dentro da mesma
transação SQL** que já atualiza o pedido e o histórico — "se a transação principal commitou, o evento foi
registrado, e se ela deu rollback, o evento some junto" (`[09:06] Diego`). O trade-off é explícito: se a
gravação do evento falhar, a mudança de status **também** falha — "Não pode ter caso de status mudar e
evento não sair" (`[09:40] Bruno`). Ver [ADR-001](./adrs/ADR-001-outbox-no-mysql.md).

**b) Desacoplamento por processo separado em polling.** Um worker roda **fora da instância da API** —
"senão se a API reinicia, perde o worker" (`[09:11] Diego`) — e verifica a outbox a cada 2 segundos,
processando os pendentes mais antigos em lotes pequenos. Dois segundos cabem folgadamente nos 10
exigidos (`[09:10] Larissa`). Banco e stack são compartilhados com a API; o processo, nunca. Ver
[ADR-005](./adrs/ADR-005-worker-separado-em-polling.md).

**c) Resiliência por retentativa espaçada, com destino final.** Cinco tentativas com espaçamento
crescente somando cerca de 15 horas; esgotadas, o evento vai para uma **tabela de dead letter** com o
motivo da falha, que serve de evidência para debug e de origem para reprocessamento manual por
administrador (`[09:18] Diego`). Ver [ADR-002](./adrs/ADR-002-retry-com-backoff-e-dlq.md).

**d) Confiança por assinatura, com raio de dano contido.** Expomos dados de pedidos a um endpoint fora da
nossa infraestrutura, e o cliente precisa poder verificar origem e integridade (`[09:19] Sofia`).
Assinamos com **HMAC-SHA256** usando uma **credencial exclusiva por endpoint** — "senão se vaza uma, vaza
tudo" (`[09:21] Sofia`) — rotacionável com 24 h de convivência. Ver
[ADR-003](./adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md).

**e) Semântica de entrega assumida, não escondida.** A entrega é **at-least-once**: o cliente pode receber
o mesmo evento duas vezes e deduplica pelo identificador único que enviamos. Ver
[ADR-004](./adrs/ADR-004-entrega-at-least-once-com-x-event-id.md).

### 3.3 Duas escolhas de modelagem que sustentam o resto

- **Snapshot na inserção.** O evento guarda o payload **já renderizado** no momento da transição, não a
  referência ao pedido — assim a notificação continua fiel ao momento que descreve mesmo que o pedido
  mude antes da entrega (`[09:52] Larissa`, ratificado por `[09:52] Diego` e `[09:52] Bruno`). Ver
  [ADR-007](./adrs/ADR-007-snapshot-do-payload-na-insercao.md).
- **Filtragem na origem.** Cada endpoint assina uma lista de status, e a filtragem acontece **na
  inserção**: se nenhum endpoint do cliente quer aquela transição, o evento nem é gravado
  (`[09:34] Bruno`, `[09:34] Diego`). Ver
  [ADR-008](./adrs/ADR-008-filtragem-de-eventos-na-insercao.md).

### 3.4 Encaixe na codebase

A feature entra como um módulo comum, seguindo a estrutura já usada por todos os domínios do OMS, e
**não introduz nenhuma dependência ou padrão novo** — reaproveita classes de erro, middleware de erro
centralizado, logger, guard de papel e padrão de validação. Ver
[ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes.md); os pontos de integração, arquivo a arquivo,
estão no [FDD](./FDD.md). O único elemento estrutural novo é a **segunda entry-point** do projeto, para o
processo do worker (`[09:11] Larissa`).

## 4. Alternativas consideradas

| # | Alternativa | Trade-off que motivou o descarte | Origem |
| --- | --- | --- | --- |
| **RFC-ALT-01** | **Disparo síncrono** da chamada HTTP dentro da operação de mudança de status. | **Acoplaria a disponibilidade do nosso fluxo de pedidos à de terceiros.** Um cliente lento travaria a mudança de status de outros pedidos, e com o cliente fora do ar não haveria saída boa — "o que a gente faz, dá rollback na mudança de status? Não dá". | `[09:04] Bruno`; `[09:06] Diego` ("Síncrono está fora de questão") |
| **RFC-ALT-02** | **Redis Streams** ou mensageria dedicada, em vez de outbox no MySQL existente. | **Custo operacional permanente desproporcional ao ganho.** Com um time pequeno, "subir Redis Cluster pra isso é overengineering"; o MySQL existente resolve o mesmo problema. | `[09:07] Larissa` (levanta); `[09:07] Diego` (descarta) |
| **RFC-ALT-03** | **Trigger de banco** notificando o worker, em vez de polling. | **O mecanismo não existe no MySQL e o substituto seria uma gambiarra.** Não há listener nativo equivalente ao `NOTIFY`/`LISTEN` do Postgres; a trigger só executa SQL. Avisar o worker exigiria improviso, e o polling de 2 s já atende os 10 s com folga. | `[09:09] Bruno` (propõe); `[09:09] Diego` (descarta) |
| **RFC-ALT-04** | **Entrega exactly-once**, em vez de at-least-once com dedup no cliente. | **Complexidade desproporcional ao problema real:** exigiria coordenação dos dois lados. At-least-once "resolve 99% dos casos" e é o padrão de mercado (Stripe, GitHub). O custo foi nomeado por Sofia: "isso joga responsabilidade pro cliente". | `[09:25] Sofia` (levanta o custo); `[09:25] Diego` (decide) |
| **RFC-ALT-05** | **3 tentativas** (mais agressivo) ou **retentativa indefinida** (extremo oposto). | **Recusado nas duas pontas.** Três cobririam só ~30 min e matariam o evento de um cliente com indisponibilidade matinal — já houve cliente com 2 h de manutenção planejada. Indefinida deixa evento pendurado para sempre se o cliente sumiu. Cinco cobrem 12–24 h e terminam. | `[09:15] Diego` (contra indefinido); `[09:16] Bruno` (propõe 3); `[09:16] Diego` (descarta) |
| **RFC-ALT-06** | **Marcar `failed` na própria outbox**, em vez de tabela de dead letter separada. | **Legibilidade operacional:** a tabela separada mantém a leitura da outbox limpa e isola a evidência para debug e reprocessamento. | `[09:17] Larissa` (coloca as opções); `[09:18] Diego` (decide) |
| **RFC-ALT-07** | **Guardar só a referência ao pedido** e renderizar o payload no envio. | **O evento deixaria de ser fiel ao momento que descreve:** se o pedido mudasse antes da entrega, o cliente receberia uma transição antiga com dados novos. | `[09:51] Bruno` (levanta); `[09:52] Larissa` (decide) |

## 5. Questões em aberto

| # | Questão | Estado | Origem |
| --- | --- | --- | --- |
| **RFC-Q-01** | **Rate limiting de saída.** Um cliente com 50 pedidos mudando de status em um minuto recebe 50 chamadas nossas. Limitamos a vazão por destino? | **Não decidido, deliberadamente.** Fora do escopo desta fase; a postura acordada é "observar e implementar se virar problema". Reabrir com dados de produção. | `[09:38] Diego`; `[09:39] Larissa` |
| **RFC-Q-02** | **Alerta ao cliente quando o webhook dele está falhando.** Hoje ele só descobre consultando o histórico de entregas. | **Adiado**, explicitamente condicionado a medição: "talvez próxima fase, depois que a gente medir o impacto". | `[09:37] Marcos`; `[09:37] Larissa` |
| **RFC-Q-03** | **Estratégia de escala do processamento.** O worker único é o que sustenta a ordenação por pedido; como escalar quando o volume exigir mais? | **Adiado.** Dois caminhos foram nomeados — particionar por pedido ou lock pessimista — sem escolha: "isso é problema do futuro, não agora". | `[09:13] Bruno`; `[09:13] Diego`; `[09:13] Larissa` |
| **RFC-Q-04** | **Retenção e expurgo dos eventos entregues.** A outbox cresce indefinidamente sem arquivamento. | **Fora do escopo desta feature, por decisão explícita.** A necessidade foi reconhecida e a ordem de grandeza citada como ilustração ("30 dias ou assim"), sem prazo decidido. | `[09:08] Diego` |
| **RFC-Q-05** | **Endurecimento da autorização do CRUD de configuração.** Hoje qualquer usuário autenticado a mantém; só o replay exige administrador. | **Adiado conscientemente:** "Por enquanto sim. Mais pra frente a gente pode endurecer." | `[09:37] Marcos`; `[09:37] Sofia` |

## 6. Impacto e riscos

### Impacto sobre o sistema existente

| Área | Impacto |
| --- | --- |
| **Fluxo de mudança de status do pedido** | **Alto e deliberado** — é o único ponto do código existente que muda de comportamento. A extensão é feita por uma função que recebe a transação em curso, em vez de injetar o módulo de webhooks dentro do serviço de pedidos (`[09:41] Bruno`, `[09:41] Diego`). |
| **Demais módulos do OMS** | **Nenhum.** Módulo novo, sem tocar autenticação, produtos, clientes ou usuários. |
| **Banco de dados** | **Aditivo.** Tabelas novas; nenhuma existente é alterada. |
| **Operação** | **Um artefato novo a publicar e monitorar** — o worker, com ciclo de vida independente da API. |
| **Contrato com o cliente** | **Novo.** Exige endpoint sobre TLS, verificação da assinatura e dedup por identificador de evento. |

### Riscos arquiteturais

| # | Risco | Mitigação | Origem |
| --- | --- | --- | --- |
| **RFC-R-01** | **A atomicidade tem duas faces:** amarrar o evento à transação do pedido faz um defeito na gravação do evento derrubar a mudança de status, o fluxo mais crítico do OMS. | Trade-off aceito. Mitigado por manter no caminho crítico apenas escrita local ao banco, sem chamada externa, e por testar a transação nos dois sentidos. | `[09:40] Bruno`; `[09:41] Diego` |
| **RFC-R-02** | **A ordenação depende de premissa operacional**, não do desenho: vale enquanto houver um único worker. Escalar sem tratar isso a quebra silenciosamente. | Documentar como limitação e tratar a escala como decisão futura consciente (RFC-Q-03). Os clientes nunca pediram ordenação global (`[09:14] Marcos`). | `[09:12] Diego`; `[09:13] Larissa` |
| **RFC-R-03** | **A deduplicação vive fora do nosso controle:** a correção do resultado depende de o cliente implementar algo que não podemos verificar. | Identificador único e estável por evento, mais documentação destacada no portal, assumida pelo PM. | `[09:25] Sofia`; `[09:26] Marcos` |
| **RFC-R-04** | **A outbox cresce sem limite** enquanto não houver retenção, degradando a leitura de pendentes. | Índices por status do evento e por data de criação, com leitura em lotes pequenos, sustentam o curto prazo; retenção fica em aberto (RFC-Q-04). | `[09:07] Bruno` (levanta); `[09:08] Diego` (índices e batch) |
| **RFC-R-05** | **Sem rate limiting de saída, a plataforma pode sobrecarregar o próprio cliente** num pico de mudanças. | Nenhuma nesta fase — risco aceito e monitorado (RFC-Q-01). | `[09:38] Diego`; `[09:39] Larissa` |

### Impacto de prazo

Estimativa de **3 sprints**: modelagem de outbox e dead letter (1); worker e retentativas (1); CRUD de
configuração e histórico de entregas (½); integração no fluxo de pedidos e testes ponta a ponta (½);
assinatura, schemas e validações (o restante) (`[09:46] Larissa`). A **revisão de segurança é
bloqueante** e exige no mínimo 2 dias úteis antes do deploy (`[09:46] Sofia`).

## 7. Decisões relacionadas

| ADR | Decisão |
| --- | --- |
| [ADR-001 — Padrão outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md) | Registrar o evento na mesma transação da mudança de status, no MySQL existente *(§3.2a, RFC-ALT-01/02)* |
| [ADR-002 — Retry com backoff e DLQ](./adrs/ADR-002-retry-com-backoff-e-dlq.md) | 5 tentativas em 1m/5m/30m/2h/12h, depois dead letter com replay manual *(§3.2c, RFC-ALT-05/06)* |
| [ADR-003 — HMAC-SHA256 com secret por endpoint](./adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md) | Assinatura do corpo com credencial exclusiva por endpoint, rotacionável com 24 h de convivência *(§3.2d)* |
| [ADR-004 — Entrega at-least-once com `X-Event-Id`](./adrs/ADR-004-entrega-at-least-once-com-x-event-id.md) | Assumir duplicidade possível e deduplicar no cliente por identificador de evento *(§3.2e, RFC-ALT-04)* |
| [ADR-005 — Worker separado em polling](./adrs/ADR-005-worker-separado-em-polling.md) | Processo separado da API, consumindo a outbox a cada 2 segundos *(§3.2b, RFC-ALT-03)* |
| [ADR-006 — Reuso dos padrões existentes](./adrs/ADR-006-reuso-dos-padroes-existentes.md) | Módulo convencional reaproveitando erros, logger, middlewares e validação já existentes *(§3.4)* |
| [ADR-007 — Snapshot do payload na inserção](./adrs/ADR-007-snapshot-do-payload-na-insercao.md) | Gravar o payload já renderizado, para o evento permanecer fiel ao momento da transição *(§3.3, RFC-ALT-07)* |
| [ADR-008 — Filtragem de eventos na inserção](./adrs/ADR-008-filtragem-de-eventos-na-insercao.md) | Não gravar evento que nenhum endpoint ativo do cliente assina *(§3.3)* |
