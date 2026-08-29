# Tracker de Rastreabilidade

Referência cruzada entre cada item registrado no pacote de design docs e sua origem — a transcrição da
reunião (`TRANSCRICAO.md`) ou o código da aplicação.

**Como ler.** `Fonte = TRANSCRICAO` → **Localização** traz `[hh:mm] Nome`, dentro do intervalo `[09:00]`
a `[09:53]`. `Fonte = CODIGO` → **Localização** traz um caminho real, verificado no repositório.

**Regra que este documento impõe ao pacote.** Item cuja Localização não pode ser preenchida é
alucinação: volta ao documento de origem para correção ou remoção. Não há célula vazia nem "N/A" nesta
tabela.

| Documento | Itens rastreados | Fonte TRANSCRICAO | Fonte CODIGO |
| --- | --- | --- | --- |
| [`docs/PRD.md`](./PRD.md) | 91 | 86 | 5 |
| [`docs/RFC.md`](./RFC.md) | 22 | 21 | 1 |
| [`docs/adrs/`](./adrs/) | 8 | 8 | 0 |
| [`docs/FDD.md`](./FDD.md) | 126 | 89 | 37 |
| **Total** | **247** | **204 (82,6%)** | **43 (17,4%)** |

Os 43 itens com fonte `CODIGO` apontam para **21 arquivos distintos** do repositório, todos verificados.

---

## PRD — `docs/PRD.md`

### Objetivos e métricas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OBJ-01 | docs/PRD.md | Métrica | Latência de notificação abaixo de 10 segundos | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OBJ-02 | docs/PRD.md | Métrica | 5 tentativas de entrega distribuídas em janela de ~15 h | TRANSCRICAO | `[09:17] Diego` |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Zero acoplamento síncrono com o fluxo de mudança de status | TRANSCRICAO | `[09:04] Bruno` |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | 100% dos envios assinados, com credencial exclusiva por endpoint | TRANSCRICAO | `[09:21] Sofia` |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Entrega até fim de novembro, esforço de 3 sprints | TRANSCRICAO | `[09:45] Marcos` |

### Escopo — fora de escopo

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-OUT-01 | docs/PRD.md | Exclusão | E-mail de aviso ao cliente com webhook falhando — adiado | TRANSCRICAO | `[09:37] Larissa` |
| PRD-OUT-02 | docs/PRD.md | Exclusão | Painel visual para o cliente — projeto separado | TRANSCRICAO | `[09:40] Larissa` |
| PRD-OUT-03 | docs/PRD.md | Exclusão | Rate limiting de saída — observar antes de implementar | TRANSCRICAO | `[09:39] Larissa` |
| PRD-OUT-04 | docs/PRD.md | Exclusão | Webhooks inbound — escopo é apenas outbound | TRANSCRICAO | `[09:02] Marcos` |
| PRD-OUT-05 | docs/PRD.md | Exclusão | Arquivamento de eventos entregues — fora desta feature | TRANSCRICAO | `[09:08] Diego` |
| PRD-OUT-06 | docs/PRD.md | Exclusão | Endurecer autorização do CRUD de configuração — adiado | TRANSCRICAO | `[09:37] Sofia` |
| PRD-OUT-07 | docs/PRD.md | Exclusão | Escala horizontal do processamento — problema do futuro | TRANSCRICAO | `[09:13] Diego` |
| PRD-OUT-08 | docs/PRD.md | Exclusão | Documentação no portal do desenvolvedor — trabalho do PM | TRANSCRICAO | `[09:40] Marcos` |

### Requisitos funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | docs/PRD.md | Requisito Funcional | Cadastrar endpoint com URL e status assinados; secret gerada pela plataforma | TRANSCRICAO | `[09:31] Marcos` |
| PRD-FR-02 | docs/PRD.md | Requisito Funcional | Registro guarda url, secret, cliente e estado ativo | TRANSCRICAO | `[09:21] Bruno` |
| PRD-FR-03 | docs/PRD.md | Requisito Funcional | Alterar, remover e listar endpoints de um cliente | TRANSCRICAO | `[09:33] Bruno` |
| PRD-FR-04 | docs/PRD.md | Requisito Funcional | Endpoint assina quais mudanças de status quer receber | TRANSCRICAO | `[09:33] Marcos` |
| PRD-FR-05 | docs/PRD.md | Requisito Funcional | Rotação de secret com 24 h de convivência da anterior | TRANSCRICAO | `[09:21] Sofia` |
| PRD-FR-06 | docs/PRD.md | Requisito Funcional | Toda mudança de status gera evento atômico com a mudança | TRANSCRICAO | `[09:40] Bruno` |
| PRD-FR-07 | docs/PRD.md | Requisito Funcional | Notificação descreve a transição; sem composição de itens | TRANSCRICAO | `[09:43] Diego` |
| PRD-FR-08 | docs/PRD.md | Requisito Funcional | Cada notificação assinada criptograficamente | TRANSCRICAO | `[09:20] Sofia` |
| PRD-FR-09 | docs/PRD.md | Requisito Funcional | Identificador único de evento para dedup no cliente | TRANSCRICAO | `[09:25] Diego` |
| PRD-FR-10 | docs/PRD.md | Requisito Funcional | Até 5 retentativas em 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Larissa` |
| PRD-FR-11 | docs/PRD.md | Requisito Funcional | Evento falho preservado com motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| PRD-FR-12 | docs/PRD.md | Requisito Funcional | Administrador reprocessa evento morto, com autoria registrada | TRANSCRICAO | `[09:36] Sofia` |
| PRD-FR-13 | docs/PRD.md | Requisito Funcional | Cliente consulta histórico de entregas do endpoint | TRANSCRICAO | `[09:34] Marcos` |
| PRD-FR-14 | docs/PRD.md | Requisito Funcional | Cadastro recusa destino sem TLS | TRANSCRICAO | `[09:23] Sofia` |
| PRD-FR-15 | docs/PRD.md | Requisito Funcional | Evento acima de 64 KB não é enviado; erra em vez de truncar | TRANSCRICAO | `[09:24] Diego` |
| PRD-FR-16 | docs/PRD.md | Requisito Funcional | CRUD exige autenticado; replay exige administrador | TRANSCRICAO | `[09:37] Sofia` |

### Requisitos não funcionais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-NFR-01 | docs/PRD.md | Requisito Não Funcional | Latência de notificação abaixo de 10 s | TRANSCRICAO | `[09:02] Marcos` |
| PRD-NFR-02 | docs/PRD.md | Requisito Não Funcional | Mudança de status não bloqueia por cliente lento ou fora do ar | TRANSCRICAO | `[09:04] Bruno` |
| PRD-NFR-03 | docs/PRD.md | Requisito Não Funcional | Janela de retentativa cobre indisponibilidade longa (~15 h) | TRANSCRICAO | `[09:16] Diego` |
| PRD-NFR-04 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada ao destino | TRANSCRICAO | `[09:42] Diego` |
| PRD-NFR-05 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once; dedup é do cliente | TRANSCRICAO | `[09:24] Diego` |
| PRD-NFR-06 | docs/PRD.md | Requisito Não Funcional | Notificação reflete o estado do momento da transição | TRANSCRICAO | `[09:52] Larissa` |
| PRD-NFR-07 | docs/PRD.md | Requisito Não Funcional | Ordenação por pedido, sem garantia global | TRANSCRICAO | `[09:13] Larissa` |
| PRD-NFR-08 | docs/PRD.md | Requisito Não Funcional | Comunicação exclusivamente sobre TLS | TRANSCRICAO | `[09:23] Sofia` |
| PRD-NFR-09 | docs/PRD.md | Requisito Não Funcional | Credencial exclusiva por endpoint, rotacionável | TRANSCRICAO | `[09:21] Sofia` |
| PRD-NFR-10 | docs/PRD.md | Requisito Não Funcional | Limite de 64 KB por evento | TRANSCRICAO | `[09:24] Larissa` |
| PRD-NFR-11 | docs/PRD.md | Requisito Não Funcional | Nenhuma infraestrutura nova — time pequeno | TRANSCRICAO | `[09:07] Diego` |
| PRD-NFR-12 | docs/PRD.md | Requisito Não Funcional | Entrega não interrompida por reinício da API | TRANSCRICAO | `[09:11] Diego` |
| PRD-NFR-13 | docs/PRD.md | Requisito Não Funcional | Intervenção manual na fila registra o autor | TRANSCRICAO | `[09:36] Sofia` |

### Dependências

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-DEP-01 | docs/PRD.md | Dependência | Fluxo de mudança de status do pedido é a origem de todo evento | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-DEP-02 | docs/PRD.md | Dependência | Banco MySQL existente; nada provisionado a mais | CODIGO | `prisma/schema.prisma` |
| PRD-DEP-03 | docs/PRD.md | Dependência | Autenticação e controle de papel já existentes | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-DEP-04 | docs/PRD.md | Dependência | Novo processo operacional a publicar e monitorar | TRANSCRICAO | `[09:11] Diego` |
| PRD-DEP-05 | docs/PRD.md | Dependência | Revisão de segurança bloqueante, mínimo 2 dias úteis | TRANSCRICAO | `[09:46] Sofia` |
| PRD-DEP-06 | docs/PRD.md | Dependência | Sessão de revisão do design antes de codar | TRANSCRICAO | `[09:50] Larissa` |
| PRD-DEP-07 | docs/PRD.md | Dependência | Cliente precisa expor TLS, verificar assinatura e deduplicar | TRANSCRICAO | `[09:25] Diego` |

### Riscos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-RSK-01 | docs/PRD.md | Risco | Cliente não deduplica e processa evento em duplicidade | TRANSCRICAO | `[09:25] Sofia` |
| PRD-RSK-02 | docs/PRD.md | Risco | Vazamento da secret pelo lado do cliente — precedente real | TRANSCRICAO | `[09:22] Diego` |
| PRD-RSK-03 | docs/PRD.md | Risco | Cliente indisponível além de ~15 h perde eventos | TRANSCRICAO | `[09:17] Marcos` |
| PRD-RSK-04 | docs/PRD.md | Risco | Rajada de chamadas sobrecarrega o próprio cliente | TRANSCRICAO | `[09:38] Diego` |
| PRD-RSK-05 | docs/PRD.md | Risco | Perda da conta Atlas se a entrega atrasar | TRANSCRICAO | `[09:00] Marcos` |
| PRD-RSK-06 | docs/PRD.md | Risco | Entregas fora de ordem se o processamento for paralelizado | TRANSCRICAO | `[09:12] Diego` |
| PRD-RSK-07 | docs/PRD.md | Risco | Falha ao registrar evento impede a mudança de status | TRANSCRICAO | `[09:40] Bruno` |

### Contexto, público e trade-offs

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Restrição | Demanda formal de 3 clientes B2B: Atlas, MaxDistribuição, Nova Cargo | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-02 | docs/PRD.md | Restrição | Polling atual torna a integração lenta e cara para o cliente | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-03 | docs/PRD.md | Restrição | Atlas pode migrar ao concorrente se não sair no trimestre | TRANSCRICAO | `[09:00] Marcos` |
| PRD-CTX-04 | docs/PRD.md | Restrição | Escopo outbound apenas: cliente recebe, não envia | TRANSCRICAO | `[09:02] Marcos` |
| PRD-CTX-05 | docs/PRD.md | Restrição | Aplicação não tem hoje eventos, filas ou notificação externa | CODIGO | `src/routes/index.ts` |
| PRD-CTX-06 | docs/PRD.md | Público-alvo | Usuários autenticados do OMS representam o cliente | TRANSCRICAO | `[09:32] Marcos` |
| PRD-CTX-07 | docs/PRD.md | Cenário de uso | Cliente assina apenas SHIPPED e DELIVERED | TRANSCRICAO | `[09:33] Marcos` |
| PRD-CTX-08 | docs/PRD.md | Cenário de uso | Retentativa cobre manutenção planejada de 2 h já observada | TRANSCRICAO | `[09:16] Diego` |
| PRD-TO-01 | docs/PRD.md | Trade-off | Outbox no banco existente vs. mensageria dedicada | TRANSCRICAO | `[09:07] Diego` |
| PRD-TO-02 | docs/PRD.md | Trade-off | Entrega assíncrona por processo separado, com latência mínima | TRANSCRICAO | `[09:10] Larissa` |
| PRD-TO-03 | docs/PRD.md | Trade-off | 5 tentativas: cobre indisponibilidade longa, mas atrasa até 15 h | TRANSCRICAO | `[09:17] Larissa` |
| PRD-TO-04 | docs/PRD.md | Trade-off | Secret por endpoint: contém dano, custa gestão de ciclo de vida | TRANSCRICAO | `[09:21] Sofia` |
| PRD-TO-05 | docs/PRD.md | Trade-off | At-least-once: simples para nós, exige trabalho do cliente | TRANSCRICAO | `[09:25] Diego` |
| PRD-TO-06 | docs/PRD.md | Trade-off | Reuso dos padrões: velocidade ao custo de herdar limitações | TRANSCRICAO | `[09:30] Larissa` |
| PRD-TO-07 | docs/PRD.md | Trade-off | Snapshot: fidelidade ao momento vs. dado possivelmente antigo | TRANSCRICAO | `[09:52] Larissa` |
| PRD-TO-08 | docs/PRD.md | Trade-off | Filtro na origem: economia vs. evento anterior nunca alcançável | TRANSCRICAO | `[09:34] Bruno` |

### Critérios de aceitação

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-AC-01 | docs/PRD.md | Critério de Aceitação | CRUD de endpoints funciona e devolve a secret na criação | TRANSCRICAO | `[09:31] Marcos` |
| PRD-AC-02 | docs/PRD.md | Critério de Aceitação | Destino sem TLS é recusado | TRANSCRICAO | `[09:23] Sofia` |
| PRD-AC-03 | docs/PRD.md | Critério de Aceitação | Notificação entregue em menos de 10 s no caminho normal | TRANSCRICAO | `[09:02] Marcos` |
| PRD-AC-04 | docs/PRD.md | Critério de Aceitação | Transição sem assinante não gera evento | TRANSCRICAO | `[09:34] Bruno` |
| PRD-AC-05 | docs/PRD.md | Critério de Aceitação | Falha ao registrar evento impede a mudança de status | TRANSCRICAO | `[09:41] Diego` |
| PRD-AC-06 | docs/PRD.md | Critério de Aceitação | Assinatura verificável; alteração no corpo invalida | TRANSCRICAO | `[09:20] Sofia` |
| PRD-AC-07 | docs/PRD.md | Critério de Aceitação | Exatamente 5 tentativas nos intervalos definidos | TRANSCRICAO | `[09:17] Larissa` |
| PRD-AC-08 | docs/PRD.md | Critério de Aceitação | Destino que não responde em 10 s é falha | TRANSCRICAO | `[09:42] Diego` |
| PRD-AC-09 | docs/PRD.md | Critério de Aceitação | Replay exige ADMIN e registra o autor | TRANSCRICAO | `[09:36] Sofia` |
| PRD-AC-10 | docs/PRD.md | Critério de Aceitação | Histórico de entregas consultável pelo cliente | TRANSCRICAO | `[09:34] Marcos` |
| PRD-AC-11 | docs/PRD.md | Critério de Aceitação | Rotação mantém a secret anterior por 24 h | TRANSCRICAO | `[09:21] Sofia` |
| PRD-AC-12 | docs/PRD.md | Critério de Aceitação | Evento acima de 64 KB não é enviado | TRANSCRICAO | `[09:24] Diego` |
| PRD-AC-13 | docs/PRD.md | Critério de Aceitação | Notificação reflete o estado do momento da transição | TRANSCRICAO | `[09:52] Larissa` |
| PRD-AC-14 | docs/PRD.md | Critério de Aceitação | Reinício da API não interrompe entregas pendentes | TRANSCRICAO | `[09:11] Diego` |
| PRD-AC-15 | docs/PRD.md | Critério de Aceitação | Revisão de segurança concluída antes do deploy | TRANSCRICAO | `[09:49] Sofia` |

### Estratégia de testes

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-TST-01 | docs/PRD.md | Estratégia de Teste | Meia sprint reservada a integração e testes ponta a ponta | TRANSCRICAO | `[09:46] Larissa` |
| PRD-TST-02 | docs/PRD.md | Estratégia de Teste | Revisão de segurança manual sobre HMAC e geração de secret | TRANSCRICAO | `[09:46] Sofia` |
| PRD-TST-03 | docs/PRD.md | Estratégia de Teste | Suíte existente Vitest + Supertest é a base dos testes | CODIGO | `tests/helpers/factories.ts` |
| PRD-TST-04 | docs/PRD.md | Estratégia de Teste | Validação com o cliente: parar de fazer polling | TRANSCRICAO | `[09:02] Marcos` |

---

## RFC — `docs/RFC.md`

### Alternativas consideradas

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-ALT-01 | docs/RFC.md | Alternativa Descartada | Disparo síncrono dentro da transação de mudança de status | TRANSCRICAO | `[09:06] Diego` |
| RFC-ALT-02 | docs/RFC.md | Alternativa Descartada | Redis Streams ou mensageria dedicada | TRANSCRICAO | `[09:07] Diego` |
| RFC-ALT-03 | docs/RFC.md | Alternativa Descartada | Trigger de banco notificando o worker | TRANSCRICAO | `[09:09] Diego` |
| RFC-ALT-04 | docs/RFC.md | Alternativa Descartada | Entrega exactly-once | TRANSCRICAO | `[09:25] Diego` |
| RFC-ALT-05 | docs/RFC.md | Alternativa Descartada | 3 tentativas, ou retentativa indefinida | TRANSCRICAO | `[09:16] Diego` |
| RFC-ALT-06 | docs/RFC.md | Alternativa Descartada | Marcar failed na própria outbox em vez de tabela separada | TRANSCRICAO | `[09:18] Diego` |
| RFC-ALT-07 | docs/RFC.md | Alternativa Descartada | Guardar só a referência ao pedido e renderizar no envio | TRANSCRICAO | `[09:52] Larissa` |

### Questões em aberto

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-Q-01 | docs/RFC.md | Questão em Aberto | Rate limiting de saída — observar e decidir depois | TRANSCRICAO | `[09:39] Larissa` |
| RFC-Q-02 | docs/RFC.md | Questão em Aberto | Alerta ao cliente com webhook falhando — próxima fase | TRANSCRICAO | `[09:37] Larissa` |
| RFC-Q-03 | docs/RFC.md | Questão em Aberto | Estratégia de escala: partição por pedido ou lock pessimista | TRANSCRICAO | `[09:13] Diego` |
| RFC-Q-04 | docs/RFC.md | Questão em Aberto | Retenção e expurgo dos eventos entregues | TRANSCRICAO | `[09:08] Diego` |
| RFC-Q-05 | docs/RFC.md | Questão em Aberto | Endurecer autorização do CRUD de configuração | TRANSCRICAO | `[09:37] Sofia` |

### Impacto e riscos arquiteturais

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| RFC-R-01 | docs/RFC.md | Risco | Atomicidade tem duas faces: defeito no evento derruba o pedido | TRANSCRICAO | `[09:41] Diego` |
| RFC-R-02 | docs/RFC.md | Risco | Ordenação é premissa operacional, não garantia do desenho | TRANSCRICAO | `[09:12] Diego` |
| RFC-R-03 | docs/RFC.md | Risco | Deduplicação vive fora do nosso controle | TRANSCRICAO | `[09:25] Sofia` |
| RFC-R-04 | docs/RFC.md | Risco | Outbox cresce sem limite sem política de retenção | TRANSCRICAO | `[09:07] Bruno` |
| RFC-R-05 | docs/RFC.md | Risco | Sem rate limiting, a plataforma pode sobrecarregar o cliente | TRANSCRICAO | `[09:38] Diego` |
| RFC-IMP-01 | docs/RFC.md | Restrição | Único ponto do código existente que muda de comportamento | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-IMP-02 | docs/RFC.md | Restrição | Estimativa de 3 sprints com revisão de segurança inclusa | TRANSCRICAO | `[09:47] Larissa` |
| RFC-IMP-03 | docs/RFC.md | Restrição | Extensão por função que recebe o tx, sem injetar repository | TRANSCRICAO | `[09:41] Diego` |
| RFC-IMP-04 | docs/RFC.md | Restrição | Snapshot na inserção e filtragem na origem | TRANSCRICAO | `[09:34] Bruno` |
| RFC-META-01 | docs/RFC.md | Decisão | Revisores são os cinco participantes da reunião | TRANSCRICAO | `[09:49] Diego` |

---

## ADRs — `docs/adrs/`

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão outbox no MySQL, evento na mesma transação | TRANSCRICAO | `[09:08] Larissa` |
| ADR-002 | docs/adrs/ADR-002-retry-com-backoff-e-dlq.md | Decisão | 5 tentativas em backoff 1m/5m/30m/2h/12h, depois DLQ | TRANSCRICAO | `[09:17] Larissa` |
| ADR-003 | docs/adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256, secret por endpoint, rotação com 24 h | TRANSCRICAO | `[09:22] Sofia` |
| ADR-004 | docs/adrs/ADR-004-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id para dedup no cliente | TRANSCRICAO | `[09:26] Larissa` |
| ADR-005 | docs/adrs/ADR-005-worker-separado-em-polling.md | Decisão | Worker em processo separado, polling de 2 segundos | TRANSCRICAO | `[09:11] Diego` |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo dos padrões do projeto, nada novo | TRANSCRICAO | `[09:30] Larissa` |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Payload renderizado como snapshot na inserção | TRANSCRICAO | `[09:52] Larissa` |
| ADR-008 | docs/adrs/ADR-008-filtragem-de-eventos-na-insercao.md | Decisão | Filtrar por assinatura na inserção, não no envio | TRANSCRICAO | `[09:34] Bruno` |

---

## FDD (TDD) — `docs/FDD.md`

### Objetivos técnicos e fluxos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OT-01 | docs/FDD.md | Objetivo Técnico | Registrar evento na transação sem I/O externa | TRANSCRICAO | `[09:40] Bruno` |
| FDD-OT-02 | docs/FDD.md | Objetivo Técnico | Entregar em menos de 10 s, piso de 2 s | TRANSCRICAO | `[09:10] Larissa` |
| FDD-OT-03 | docs/FDD.md | Objetivo Técnico | Executar backoff com teto de 5 e destino em DLQ | TRANSCRICAO | `[09:17] Diego` |
| FDD-OT-04 | docs/FDD.md | Objetivo Técnico | Assinar com HMAC sem registrar a secret em log | TRANSCRICAO | `[09:21] Sofia` |
| FDD-OT-05 | docs/FDD.md | Objetivo Técnico | Expor API seguindo os padrões existentes | CODIGO | `src/modules/orders/order.routes.ts` |
| FDD-OT-06 | docs/FDD.md | Objetivo Técnico | Consumo em processo separado da API | TRANSCRICAO | `[09:11] Diego` |
| FDD-FLW-01 | docs/FDD.md | Fluxo | Criação do evento na outbox dentro da transação | TRANSCRICAO | `[09:06] Diego` |
| FDD-FLW-02 | docs/FDD.md | Fluxo | Loop do worker: lê pendentes em batch, processa, marca | TRANSCRICAO | `[09:09] Diego` |
| FDD-FLW-03 | docs/FDD.md | Fluxo | Retentativa: calcula nextAttemptAt pela progressão | TRANSCRICAO | `[09:17] Diego` |
| FDD-FLW-04 | docs/FDD.md | Fluxo | Dead letter e replay preservando o event_id | TRANSCRICAO | `[09:18] Diego` |
| FDD-FLW-05 | docs/FDD.md | Fluxo | Rotação de secret com janela de convivência | TRANSCRICAO | `[09:21] Sofia` |

### Modelo de dados

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-DAT-01 | docs/FDD.md | Restrição | webhook_endpoints guarda url, secret, cliente e ativo | TRANSCRICAO | `[09:21] Bruno` |
| FDD-DAT-02 | docs/FDD.md | Restrição | Estados da outbox: pendente, processando, falhou, entregue | TRANSCRICAO | `[09:08] Diego` |
| FDD-DAT-03 | docs/FDD.md | Restrição | Índices em status do evento e created_at, leitura em batch | TRANSCRICAO | `[09:08] Diego` |
| FDD-DAT-04 | docs/FDD.md | Restrição | webhook_deliveries guarda payload, response e duração | TRANSCRICAO | `[09:34] Marcos` |
| FDD-DAT-05 | docs/FDD.md | Restrição | webhook_dead_letter guarda payload, motivo e timestamp | TRANSCRICAO | `[09:18] Diego` |
| FDD-DAT-06 | docs/FDD.md | Restrição | Chave primária em UUID, padrão do projeto | CODIGO | `prisma/schema.prisma` |
| FDD-DAT-07 | docs/FDD.md | Restrição | OrderStatusHistory é o precedente de modelagem | CODIGO | `prisma/schema.prisma` |
| FDD-DAT-08 | docs/FDD.md | Restrição | subscribedStatuses restrito ao enum OrderStatus | CODIGO | `prisma/schema.prisma` |

### Contratos públicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks — cadastro, 201, secret exibida uma vez | TRANSCRICAO | `[09:31] Marcos` |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks — listagem paginada por cliente, 200 | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id — alteração, 200 | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id — remoção, 204 | TRANSCRICAO | `[09:33] Bruno` |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /webhooks/:id/rotate-secret — rotação, 200 | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries — histórico, 200 | TRANSCRICAO | `[09:34] Marcos` |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay — 202, ADMIN | TRANSCRICAO | `[09:35] Diego` |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Prefixo /api/v1 e registro via buildApiRouter | CODIGO | `src/routes/index.ts` |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Formato de listagem { data, pagination } | CODIGO | `src/shared/http/response.ts` |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Payload do evento: event_id, event_type, timestamp, order, status | TRANSCRICAO | `[09:43] Diego` |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Payload não inclui items; detalhe via GET /orders/:id | TRANSCRICAO | `[09:43] Diego` |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Headers X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | `[09:44] Diego` |
| FDD-CONTRATO-13 | docs/FDD.md | Contrato | Header X-Webhook-Id identifica o cadastro de origem | TRANSCRICAO | `[09:44] Sofia` |
| FDD-CONTRATO-14 | docs/FDD.md | Contrato | customerId vem do corpo, não do JWT | TRANSCRICAO | `[09:32] Larissa` |
| FDD-CONTRATO-15 | docs/FDD.md | Contrato | Formato de erro { error: { code, message, details? } } | CODIGO | `src/middlewares/error.middleware.ts` |

### Matriz de erros

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND — 404, endpoint inexistente | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERR-02 | docs/FDD.md | Erro | WEBHOOK_INVALID_URL — 400, URL ausente ou sem TLS | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_SECRET_REQUIRED — 400, sem secret vigente | TRANSCRICAO | `[09:28] Bruno` |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_CUSTOMER_NOT_FOUND — 404, customerId inexistente | TRANSCRICAO | `[09:32] Larissa` |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_DUPLICATE_URL — 409, URL repetida no cliente | TRANSCRICAO | `[09:21] Bruno` |
| FDD-ERR-06 | docs/FDD.md | Erro | WEBHOOK_INVALID_STATUS_FILTER — 400, assinatura inválida | TRANSCRICAO | `[09:33] Marcos` |
| FDD-ERR-07 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE — 422, acima de 64 KB | TRANSCRICAO | `[09:24] Diego` |
| FDD-ERR-08 | docs/FDD.md | Erro | WEBHOOK_ROTATION_IN_PROGRESS — 409, janela ainda aberta | TRANSCRICAO | `[09:21] Sofia` |
| FDD-ERR-09 | docs/FDD.md | Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND — 404 | TRANSCRICAO | `[09:35] Diego` |
| FDD-ERR-10 | docs/FDD.md | Erro | WEBHOOK_ALREADY_REPLAYED — 409 | TRANSCRICAO | `[09:18] Diego` |
| FDD-ERR-11 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT — interno, destino excedeu 10 s | TRANSCRICAO | `[09:42] Diego` |
| FDD-ERR-12 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_FAILED — interno, resposta fora de 2xx | TRANSCRICAO | `[09:15] Diego` |
| FDD-ERR-13 | docs/FDD.md | Erro | Prefixo WEBHOOK_ em todos os códigos do módulo | TRANSCRICAO | `[09:29] Larissa` |
| FDD-ERR-14 | docs/FDD.md | Erro | Classes herdam de AppError e das subclasses HTTP | CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-ERR-15 | docs/FDD.md | Erro | 401 e 403 do replay reusam Unauthorized e Forbidden | CODIGO | `src/middlewares/auth.middleware.ts` |

### Resiliência

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-RES-01 | docs/FDD.md | Restrição | Timeout de 10 s por tentativa | TRANSCRICAO | `[09:42] Diego` |
| FDD-RES-02 | docs/FDD.md | Restrição | 5 tentativas com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | `[09:17] Diego` |
| FDD-RES-03 | docs/FDD.md | Restrição | Estado terminal em dead letter, sem retentativa automática | TRANSCRICAO | `[09:18] Diego` |
| FDD-RES-04 | docs/FDD.md | Trade-off | Sem canal de fallback: e-mail ficou fora desta fase | TRANSCRICAO | `[09:37] Larissa` |
| FDD-RES-05 | docs/FDD.md | Restrição | Atomicidade: evento e status commitam ou revertem juntos | TRANSCRICAO | `[09:41] Diego` |
| FDD-RES-06 | docs/FDD.md | Restrição | Nenhuma I/O externa dentro da transação | TRANSCRICAO | `[09:04] Bruno` |
| FDD-RES-07 | docs/FDD.md | Restrição | X-Event-Id estável entre tentativas e no replay | TRANSCRICAO | `[09:25] Diego` |
| FDD-RES-08 | docs/FDD.md | Restrição | Leitura em lotes pequenos ordenada por created_at | TRANSCRICAO | `[09:08] Diego` |
| FDD-RES-09 | docs/FDD.md | Restrição | Encerramento gracioso espelhando o shutdown da API | CODIGO | `src/server.ts` |
| FDD-RES-10 | docs/FDD.md | Restrição | Recuperação de eventos travados em PROCESSING | TRANSCRICAO | `[09:11] Diego` |
| FDD-RES-11 | docs/FDD.md | Restrição | Limite de 64 KB verificado na inserção | TRANSCRICAO | `[09:24] Larissa` |
| FDD-RES-12 | docs/FDD.md | Trade-off | Sem rate limiting de saída nesta fase | TRANSCRICAO | `[09:39] Larissa` |

### Observabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-OBS-01 | docs/FDD.md | Métrica | webhook_outbox_pending_total — saúde da fila | TRANSCRICAO | `[09:07] Bruno` |
| FDD-OBS-02 | docs/FDD.md | Métrica | webhook_delivery_lag_seconds — verifica o SLA de 10 s | TRANSCRICAO | `[09:02] Marcos` |
| FDD-OBS-03 | docs/FDD.md | Métrica | webhook_delivery_total por resultado e endpoint | TRANSCRICAO | `[09:34] Marcos` |
| FDD-OBS-04 | docs/FDD.md | Métrica | webhook_delivery_duration_seconds — tempo de resposta | TRANSCRICAO | `[09:34] Marcos` |
| FDD-OBS-05 | docs/FDD.md | Métrica | webhook_delivery_attempts_total por número da tentativa | TRANSCRICAO | `[09:17] Larissa` |
| FDD-OBS-06 | docs/FDD.md | Métrica | webhook_dead_letter_total — falhas definitivas | TRANSCRICAO | `[09:18] Diego` |
| FDD-OBS-07 | docs/FDD.md | Métrica | webhook_worker_loop_duration_seconds — atraso do ciclo | TRANSCRICAO | `[09:09] Diego` |
| FDD-OBS-08 | docs/FDD.md | Log | Pino existente, nada novo de logging | TRANSCRICAO | `[09:29] Bruno` |
| FDD-OBS-09 | docs/FDD.md | Log | webhook_event_enqueued com eventId, endpointId, requestId | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-10 | docs/FDD.md | Log | webhook_delivery_attempt com status e duração | TRANSCRICAO | `[09:34] Marcos` |
| FDD-OBS-11 | docs/FDD.md | Log | webhook_dead_letter_replayed registra quem fez o replay | TRANSCRICAO | `[09:36] Sofia` |
| FDD-OBS-12 | docs/FDD.md | Log | webhook_secret_rotated nunca registra o valor da secret | TRANSCRICAO | `[09:21] Sofia` |
| FDD-OBS-13 | docs/FDD.md | Restrição | redact do Pino precisa cobrir secret, previousSecret e signature | CODIGO | `src/shared/logger/index.ts` |
| FDD-OBS-14 | docs/FDD.md | Tracing | Correlação por X-Request-Id propagado até o worker | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-OBS-15 | docs/FDD.md | Tracing | eventId correlaciona nosso lado ao do cliente | TRANSCRICAO | `[09:25] Diego` |
| FDD-OBS-16 | docs/FDD.md | Tracing | Sem tracing distribuído no projeto; lacuna consciente | CODIGO | `package.json` |

### Integração com o sistema existente

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus ganha publishWebhookEvent(tx, …) na transação | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | docs/FDD.md | Integração | Máquina de estados consultada, não alterada | CODIGO | `src/modules/orders/order.status.ts` |
| FDD-INT-03 | docs/FDD.md | Integração | Classes WEBHOOK_* herdam de AppError e subclasses | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INT-04 | docs/FDD.md | Integração | Error middleware trata os novos erros sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN') protegem as rotas | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-06 | docs/FDD.md | Integração | Precedente único de requireRole('ADMIN') no projeto | CODIGO | `src/modules/users/user.routes.ts` |
| FDD-INT-07 | docs/FDD.md | Integração | buildApiRouter e buildControllers ganham o módulo | CODIGO | `src/routes/index.ts` |
| FDD-INT-08 | docs/FDD.md | Integração | Encadeamento repository → service → controller | CODIGO | `src/app.ts` |
| FDD-INT-09 | docs/FDD.md | Integração | redactPaths do logger recebe os campos sensíveis novos | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-10 | docs/FDD.md | Integração | requestId persistido na outbox e reemitido pelo worker | CODIGO | `src/middlewares/request-logger.middleware.ts` |
| FDD-INT-11 | docs/FDD.md | Integração | Worker abre PrismaClient próprio via createPrismaClient | CODIGO | `src/config/database.ts` |
| FDD-INT-12 | docs/FDD.md | Integração | src/server.ts é o modelo da entry-point do worker | CODIGO | `src/server.ts` |
| FDD-INT-13 | docs/FDD.md | Integração | Tabelas novas seguem as convenções do schema | CODIGO | `prisma/schema.prisma` |
| FDD-INT-14 | docs/FDD.md | Integração | paginated() formata as listagens do módulo | CODIGO | `src/shared/http/response.ts` |
| FDD-INT-15 | docs/FDD.md | Integração | Validação Zod por rota via validate() | CODIGO | `src/middlewares/validate.middleware.ts` |
| FDD-INT-16 | docs/FDD.md | Integração | Testes reusam factories de Vitest + Supertest | CODIGO | `tests/helpers/factories.ts` |
| FDD-INT-17 | docs/FDD.md | Integração | Módulo em src/modules/webhooks seguindo o padrão | TRANSCRICAO | `[09:27] Bruno` |
| FDD-INT-18 | docs/FDD.md | Integração | Entry-point src/worker.ts e script npm run worker | TRANSCRICAO | `[09:11] Larissa` |
| FDD-INT-19 | docs/FDD.md | Integração | Lógica de processamento dentro do módulo de webhooks | TRANSCRICAO | `[09:28] Bruno` |

### Dependências e compatibilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-DEP-01 | docs/FDD.md | Dependência | Nenhuma biblioteca nova; crypto e fetch nativos do Node | CODIGO | `package.json` |
| FDD-DEP-02 | docs/FDD.md | Dependência | Migração Prisma aditiva, sem alterar tabela existente | CODIGO | `prisma/schema.prisma` |
| FDD-DEP-03 | docs/FDD.md | Dependência | envSchema ganha os parâmetros do worker | CODIGO | `src/config/env.ts` |
| FDD-DEP-04 | docs/FDD.md | Dependência | Worker reutiliza a mesma DATABASE_URL | TRANSCRICAO | `[09:30] Bruno` |
| FDD-DEP-05 | docs/FDD.md | Restrição | Compatibilidade retroativa total: nenhum contrato muda | TRANSCRICAO | `[09:29] Bruno` |
| FDD-DEP-06 | docs/FDD.md | Restrição | tests/orders.test.ts deve passar sem alteração | CODIGO | `tests/orders.test.ts` |

### Critérios de aceite técnicos

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-CAT-01 | docs/FDD.md | Critério de Aceite | Falha ao gravar o evento reverte a mudança de status | TRANSCRICAO | `[09:40] Bruno` |
| FDD-CAT-02 | docs/FDD.md | Critério de Aceite | Transição sem assinante não grava linha na outbox | TRANSCRICAO | `[09:34] Bruno` |
| FDD-CAT-03 | docs/FDD.md | Critério de Aceite | Suíte de pedidos existente passa sem alteração | CODIGO | `tests/orders.test.ts` |
| FDD-CAT-04 | docs/FDD.md | Critério de Aceite | Payload é snapshot e não muda com o pedido | TRANSCRICAO | `[09:52] Larissa` |
| FDD-CAT-05 | docs/FDD.md | Critério de Aceite | Acima de 64 KB dispara erro e aborta a transação | TRANSCRICAO | `[09:24] Diego` |
| FDD-CAT-06 | docs/FDD.md | Critério de Aceite | Assinatura verifica; byte alterado invalida | TRANSCRICAO | `[09:20] Sofia` |
| FDD-CAT-07 | docs/FDD.md | Critério de Aceite | Secret não aparece em resposta nem em log | TRANSCRICAO | `[09:22] Diego` |
| FDD-CAT-08 | docs/FDD.md | Critério de Aceite | Agendamento segue a progressão; 5ª falha vai para DLQ | TRANSCRICAO | `[09:17] Larissa` |
| FDD-CAT-09 | docs/FDD.md | Critério de Aceite | Destino sem resposta em 10 s é tratado como falha | TRANSCRICAO | `[09:42] Diego` |
| FDD-CAT-10 | docs/FDD.md | Critério de Aceite | X-Event-Id idêntico em todas as tentativas e no replay | TRANSCRICAO | `[09:25] Diego` |
| FDD-CAT-11 | docs/FDD.md | Critério de Aceite | Replay exige ADMIN e grava o autor | TRANSCRICAO | `[09:36] Sofia` |
| FDD-CAT-12 | docs/FDD.md | Critério de Aceite | URL http recusada no cadastro e na alteração | TRANSCRICAO | `[09:23] Sofia` |
| FDD-CAT-13 | docs/FDD.md | Critério de Aceite | Rotação mantém a anterior por 24 h e a limpa | TRANSCRICAO | `[09:21] Sofia` |
| FDD-CAT-14 | docs/FDD.md | Critério de Aceite | Reiniciar a API não interrompe entregas pendentes | TRANSCRICAO | `[09:11] Diego` |
| FDD-CAT-15 | docs/FDD.md | Critério de Aceite | Erros do módulo respondem no formato padrão com prefixo | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-CAT-16 | docs/FDD.md | Critério de Aceite | Lag de entrega abaixo de 10 s no caminho feliz | TRANSCRICAO | `[09:02] Marcos` |

### Riscos de implementação

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| FDD-R-01 | docs/FDD.md | Risco | Esquecer de estender o redact e vazar a secret em log | CODIGO | `src/shared/logger/index.ts` |
| FDD-R-02 | docs/FDD.md | Risco | Consulta de assinantes engorda a transação mais sensível | TRANSCRICAO | `[09:04] Bruno` |
| FDD-R-03 | docs/FDD.md | Risco | Worker morto sem alerta ativo; fila cresce em silêncio | TRANSCRICAO | `[09:11] Diego` |
| FDD-R-04 | docs/FDD.md | Risco | Eventos travados em PROCESSING após queda abrupta | TRANSCRICAO | `[09:11] Diego` |
| FDD-R-05 | docs/FDD.md | Risco | Segunda instância do worker quebra a ordenação em silêncio | TRANSCRICAO | `[09:13] Larissa` |
| FDD-R-06 | docs/FDD.md | Risco | Assinar corpo reserializado quebra a verificação no cliente | TRANSCRICAO | `[09:20] Sofia` |
| FDD-R-07 | docs/FDD.md | Risco | Crescimento das tabelas sem política de retenção | TRANSCRICAO | `[09:08] Diego` |
| FDD-R-08 | docs/FDD.md | Risco | Nome do arquivo do worker ficou indefinido na reunião | TRANSCRICAO | `[09:28] Bruno` |
