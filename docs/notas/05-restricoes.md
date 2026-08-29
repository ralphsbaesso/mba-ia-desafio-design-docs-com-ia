# 05 — Números, limites, prazos, SLAs e métricas citados

Recorte: todo valor quantitativo dito na reunião, com falante e timestamp. É a única fonte
autorizada para números nos documentos — **nenhum número fora desta lista pode ser usado**.

| # | Valor | O que é | Falante | Localização |
| --- | --- | --- | --- | --- |
| NUM-01 | **3 clientes B2B** — Atlas Comercial, MaxDistribuição, Nova Cargo | Origem da demanda: pedido formal recebido "na semana passada". | Marcos | `[09:00] Marcos` |
| NUM-02 | **< 10 segundos** | Definição de "tempo real" acordada com os clientes. É o SLA de latência de notificação. | Marcos | `[09:02] Marcos` |
| NUM-03 | **Fim do trimestre** | Prazo além do qual a Atlas sinalizou que pode migrar para o concorrente. | Marcos | `[09:00] Marcos` |
| NUM-04 | **Fim de novembro** | Prazo pedido pela Atlas para a entrega. | Marcos | `[09:45] Marcos` |
| NUM-05 | **3 sprints** | Estimativa de esforço, incluindo a revisão de segurança no fim. Quebra citada: modelagem outbox+DLQ (1 sprint), worker+retry (1 sprint), CRUD de configuração e deliveries (½ sprint), integração no `order.service` + testes ponta a ponta (½ sprint), HMAC/schemas/validações ("mais um pouco"). | Larissa | `[09:46] Larissa`; `[09:47] Larissa` |
| NUM-06 | **≥ 2 dias úteis** | Janela reservada para a revisão de segurança antes do deploy (HMAC e geração de secret). | Sofia | `[09:46] Sofia` |
| NUM-07 | **2 segundos** | Intervalo do loop de polling do worker; também o pior caso de latência de leitura da outbox. | Diego / Larissa | `[09:09] Diego`; `[09:10] Larissa` |
| NUM-08 | **5 tentativas** | Teto de retentativas antes de mover para a DLQ. | Diego / Larissa | `[09:15] Diego`; `[09:17] Larissa` |
| NUM-09 | **1m / 5m / 30m / 2h / 12h** | Progressão do backoff exponencial entre as 5 tentativas. | Diego | `[09:17] Diego` |
| NUM-10 | **~15 horas** | Janela total entre a primeira falha e a última tentativa, resultante da progressão acima. | Diego | `[09:17] Diego` |
| NUM-11 | **12 a 24 horas** | Janela de indisponibilidade do cliente que 5 tentativas pretendem cobrir. | Diego | `[09:15] Diego` |
| NUM-12 | **~30 minutos** | Janela que 3 tentativas cobririam — argumento usado para descartar essa opção. | Diego | `[09:16] Diego` |
| NUM-13 | **2 horas** | Indisponibilidade real já observada em cliente, em manutenção planejada — evidência a favor de 5 tentativas. | Diego | `[09:16] Diego` |
| NUM-14 | **24 horas** | Grace period durante o qual a secret antiga continua válida após rotação. | Sofia | `[09:21] Sofia` |
| NUM-15 | **10 segundos** | Timeout da chamada HTTP do worker ao endpoint do cliente. | Diego | `[09:42] Diego` |
| NUM-16 | **64 KB** | Teto de tamanho de payload por evento; acima disso, erro (não envia, não trunca). | Diego | `[09:24] Diego`; `[09:24] Larissa` |
| NUM-17 | **500 KB** | Tamanho hipotético usado por Sofia para ilustrar o problema — **não é um limite decidido**. | Sofia | `[09:23] Sofia` |
| NUM-18 | **Últimos 100 webhooks** | Volume ilustrativo do histórico de entregas que o cliente quer consultar. | Marcos | `[09:34] Marcos` |
| NUM-19 | **3 falhas seguidas** | Gatilho hipotético para o alerta por e-mail — **descartado** desta fase (ver `03-descartados.md` DESC-10). | Marcos | `[09:37] Marcos` |
| NUM-20 | **50 chamadas / minuto** | Cenário ilustrativo do problema de rate limiting de saída — **em aberto** (ver `04-adiados.md` ADI-01). | Diego | `[09:38] Diego` |
| NUM-21 | **30 dias** | Prazo ilustrativo ("ou assim") para arquivar linhas entregues da outbox — **fora do escopo** desta feature. | Diego | `[09:08] Diego` |
| NUM-22 | **99% dos casos** | Cobertura atribuída ao at-least-once com `event_id`, como argumento contra exactly-once. | Diego | `[09:25] Diego` |
| NUM-23 | **SHA-256** | Algoritmo do HMAC. | Sofia | `[09:20] Sofia` |
| NUM-24 | **~55 minutos** | Duração da reunião; intervalo válido de timestamps: `[09:00]`–`[09:53]`. | — | cabeçalho de `TRANSCRICAO.md` |

## Nota de uso

- **NUM-02** é a única métrica de sucesso quantitativa de produto disponível (gate do PRD: ≥1 objetivo
  com meta quantitativa vinda da reunião).
- **NUM-17, NUM-19, NUM-20, NUM-21** são ilustrativos ou de itens descartados/adiados: podem ser citados
  como contexto, **nunca** como requisito ou parâmetro do sistema.
