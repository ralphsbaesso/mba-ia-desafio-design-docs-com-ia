# 04 — ADIADOS para fase futura / SEM DECISÃO

Recorte: o que foi **empurrado para depois** ou ficou **em aberto** ao fim da reunião.
Alimenta "Fora de escopo" (PRD) e "Questões em aberto" (RFC). Não vira requisito.

| # | Item | Estado ao fim da reunião | Localização |
| --- | --- | --- | --- |
| ADI-01 | **Rate limiting de saída** — se o cliente tiver 50 pedidos mudando de status em um minuto, a plataforma dispara 50 chamadas nele. | **Em aberto.** Diego levantou; decidiu-se não incluir no escopo e "observar e implementar se virar problema". Registrado explicitamente como ponto em aberto. | `[09:38] Diego` (levanta); `[09:39] Diego` (fora do escopo, mas registrar); `[09:39] Larissa` ("observar e decidir depois"); ratificado em `[09:48] Larissa` |
| ADI-02 | **Notificação por e-mail ao cliente quando o webhook falha repetidamente.** | **Adiado.** Fora de escopo desta fase; possível próxima fase, "depois que a gente medir o impacto". | `[09:37] Marcos`; `[09:37] Larissa`; `[09:38] Marcos` ("anotado como futuro"); ratificado em `[09:48] Larissa` |
| ADI-03 | **Escala para múltiplos workers em paralelo** e a garantia de ordering que isso quebra. | **Adiado.** Caminhos citados: particionar por `order_id` ou lock pessimista — "problema do futuro, não agora". Fica documentado como limitação conhecida. | `[09:13] Bruno` (pergunta); `[09:13] Diego` (adia); `[09:13] Larissa` (documenta como limitação) |
| ADI-04 | **Arquivamento/expurgo das linhas entregues da outbox** (ordem de 30 dias). | **Adiado.** Diego cita "depois de 30 dias ou assim" e coloca **explicitamente fora do escopo desta feature**. O prazo é ilustrativo, não decidido. | `[09:08] Diego` |
| ADI-05 | **Endurecer a autorização do CRUD de configuração de webhook** (hoje qualquer role autenticada). | **Adiado.** "Por enquanto sim. Mais pra frente a gente pode endurecer." | `[09:37] Sofia` |
| ADI-06 | **Dashboard/painel visual para o cliente.** | **Adiado/fora de escopo.** "Agora não. Só endpoints." Projeto separado do time de frontend. | `[09:39] Marcos`; `[09:40] Larissa`; ratificado em `[09:48] Larissa` |
| ADI-07 | **Nome definitivo do arquivo de processamento do worker** — `webhook.worker.ts` ou `webhook.processor.ts`. | **Sem decisão.** Bruno propôs as duas opções ("tipo X ou Y") e Diego respondeu só "Beleza". | `[09:28] Bruno`; `[09:28] Diego` |
| ADI-08 | **Documentação para o cliente no portal do desenvolvedor** — em especial o comportamento at-least-once e como integrar via API. | **Fora do escopo técnico**, assumido pelo PM como trabalho paralelo. | `[09:26] Marcos`; `[09:40] Marcos` |
| ADI-09 | **Agendamento da revisão de segurança** (mín. 2 dias úteis, antes do deploy, sobre HMAC e geração de secret). | **Pendente de agenda.** Combinado, mas sem data; Sofia reforça no fecho da call. | `[09:46] Sofia`; `[09:47] Larissa`; `[09:49] Sofia` |
| ADI-10 | **Sessão de revisão do design doc** com Bruno e Diego antes de começar a codar. | **Pendente de agenda.** Larissa vai abrir o doc e marcar. | `[09:50] Larissa` |
