# PRD — Sistema de Webhooks de Notificação de Pedidos

| | |
| --- | --- |
| **Feature** | Sistema de Webhooks de Notificação de Pedidos |
| **Autor** | Time de Engenharia — Order Management System |
| **Status** | Aprovado para desenvolvimento |
| **Origem** | Reunião técnica de definição, `TRANSCRICAO.md` (`[09:00]`–`[09:53]`) |
| **Documentos relacionados** | [RFC](./RFC.md) · [FDD (TDD)](./FDD.md) · [ADRs](./adrs/) · [Tracker](./TRACKER.md) |

> **Altura deste documento.** O PRD responde *por que* e *o quê*. A arquitetura da solução está no
> [RFC](./RFC.md), cada decisão fechada em um [ADR](./adrs/), e o detalhe de implementação
> (contratos, payloads, códigos de erro, esquema de tabelas) no [FDD](./FDD.md). Nada aqui desce a
> esse nível.

---

## 1. Resumo e contexto da feature

Três clientes B2B da plataforma — **Atlas Comercial, MaxDistribuição e Nova Cargo** — formalizaram o
pedido de serem notificados em tempo real quando o status de seus pedidos muda no Order Management
System (`[09:00] Marcos`).

Hoje o OMS não tem nenhum mecanismo de notificação externa, evento ou fila. A única forma de o cliente
descobrir uma mudança é consultar repetidamente a API de pedidos. Esta feature preenche esse vazio com
um **sistema de webhooks outbound**: o cliente cadastra um endpoint HTTP na plataforma, escolhe quais
mudanças de status quer receber, e passa a ser chamado por nós sempre que uma delas acontece.

O escopo é **exclusivamente outbound** — da plataforma para o cliente. Os clientes querem receber
eventos, não enviá-los (`[09:02] Marcos`, `[09:02] Sofia`).

## 2. Problema e motivação

**O problema.** Os clientes fazem *polling* na API de pedidos em intervalos regulares para descobrir
mudanças de status. Marcos descreve o efeito: "isso tá deixando a integração lenta e cara pra eles"
(`[09:00] Marcos`). O custo é dos dois lados — o cliente paga por uma integração que é majoritariamente
requisição sem novidade, e a plataforma absorve esse tráfego improdutivo.

**Por que agora.** A Atlas Comercial sinalizou que, se a entrega não sair **até o fim do trimestre**,
pode migrar para um concorrente (`[09:00] Marcos`). O prazo pedido é **fim de novembro**
(`[09:45] Marcos`). A motivação, portanto, não é apenas de eficiência técnica: é **retenção de receita
de contas B2B**.

**O que "tempo real" significa aqui.** Marcos levou a pergunta aos clientes: para eles, *"qualquer coisa
abaixo de 10 segundos já é tempo real"*; o essencial é não ficar pendurado e não exigir atualização
manual (`[09:02] Marcos`). Esse número é a régua da feature.

## 3. Público-alvo e cenários de uso

### Público-alvo

| Público | Quem é | O que espera |
| --- | --- | --- |
| **Clientes B2B integradores** | Atlas Comercial, MaxDistribuição, Nova Cargo — e futuros clientes com integração via API | Ser avisado da mudança de status sem precisar consultar a plataforma |
| **Usuários da plataforma** | Usuários autenticados do OMS que representam um cliente (`[09:32] Marcos`) | Cadastrar e manter a configuração de webhooks pela API |
| **Administradores da plataforma** | Usuários com papel administrativo | Recuperar entregas que falharam definitivamente, com rastro de auditoria |
| **Time de engenharia (Pedidos e Plataforma)** | Bruno, Diego e equipe | Que a feature não degrade nem bloqueie o fluxo de pedidos existente |

### Cenários de uso

1. **Assinatura.** Um cliente registra um endpoint de destino e escolhe quais mudanças de status quer
   ouvir — por exemplo, apenas quando o pedido vira *enviado* e *entregue* (`[09:33] Marcos`). A
   plataforma gera uma credencial de assinatura e a devolve no ato do cadastro (`[09:31] Marcos`).
2. **Notificação.** O pedido muda de status no OMS. O cliente recebe, em segundos, uma chamada HTTP
   descrevendo a transição, e reage no sistema dele sem precisar consultar a plataforma.
3. **Cliente indisponível.** O endpoint do cliente está fora do ar. A plataforma **não desiste na
   primeira falha**: reprograma a entrega ao longo de uma janela de aproximadamente 15 horas
   (`[09:17] Diego`), o que cobre inclusive uma manutenção planejada de horas — cenário já observado com
   clientes reais (`[09:16] Diego`).
4. **Falha definitiva e recuperação.** Esgotadas as tentativas, o evento é preservado com o motivo da
   falha, e um administrador pode reprocessá-lo manualmente, ficando registrado quem o fez
   (`[09:18] Diego`, `[09:36] Sofia`).
5. **Auditoria pelo cliente.** O cliente consulta o histórico de entregas do seu endpoint — o que foi
   enviado, se deu certo, qual foi a resposta e quanto demorou (`[09:34] Marcos`).
6. **Troca de credencial.** O cliente suspeita de vazamento, ou faz rotação de rotina, e pede uma nova
   credencial. A anterior continua válida por 24 horas para ele migrar os sistemas sem perder eventos
   (`[09:21] Sofia`).

## 4. Objetivos e métricas de sucesso

| ID | Objetivo | Métrica | Meta | Origem |
| --- | --- | --- | --- | --- |
| **OBJ-01** | Notificar o cliente da mudança de status em tempo hábil, substituindo o polling | Latência entre a mudança de status no OMS e a entrega da notificação | **< 10 segundos** | `[09:02] Marcos` |
| **OBJ-02** | Não deixar evento se perder quando o cliente está temporariamente indisponível | Nº de tentativas de entrega antes de a plataforma desistir | **5 tentativas**, distribuídas em uma janela de **~15 horas** | `[09:17] Diego`, `[09:17] Larissa` |
| **OBJ-03** | Preservar a integridade do fluxo de pedidos existente | Impacto da feature na operação de mudança de status | **Zero** acoplamento síncrono: nenhuma chamada a sistema externo dentro da transação de pedido | `[09:04] Bruno`, `[09:06] Diego` |
| **OBJ-04** | Permitir que o cliente confie no que recebe | Cobertura de autenticidade e integridade das notificações | **100%** dos envios assinados criptograficamente, com credencial exclusiva por endpoint | `[09:20] Sofia`, `[09:21] Sofia` |
| **OBJ-05** | Reter as contas B2B que motivaram a demanda | Entrega da feature dentro do prazo negociado | **Fim de novembro**; esforço estimado em **3 sprints**, com a revisão de segurança inclusa | `[09:45] Marcos`, `[09:46] Larissa`, `[09:47] Larissa` |

> **Nota sobre as metas.** Todos os números acima vêm da reunião. Não foram definidas metas de adoção,
> de volume de eventos ou de redução percentual do tráfego de polling — a reunião não estabeleceu nenhuma,
> e defini-las aqui seria inventá-las. Ver [Questões em aberto](./RFC.md) no RFC.

## 5. Escopo

### 5.1 Incluso

- Cadastro e manutenção, pela API, dos endpoints de webhook de um cliente, com seleção de quais mudanças
  de status cada endpoint quer receber.
- Geração e rotação da credencial de assinatura, com período de convivência entre a credencial antiga e a
  nova.
- Emissão de um evento a cada mudança de status de pedido, de forma atômica com a própria mudança.
- Entrega assíncrona ao endpoint do cliente, com assinatura criptográfica do conteúdo.
- Política de retentativas com espaçamento crescente e destino final para eventos que falharam
  definitivamente.
- Reprocessamento manual, por um administrador, de eventos que falharam definitivamente, com registro de
  autoria.
- Consulta, pelo cliente, do histórico de entregas de um endpoint.

### 5.2 Fora de escopo

Cada item abaixo foi **explicitamente descartado ou adiado durante a reunião**. Nenhum deles pode ser
tratado como requisito desta fase.

| ID | Item | Situação | Origem |
| --- | --- | --- | --- |
| **OUT-01** | **Notificação por e-mail ao cliente quando o webhook dele está falhando** (ex.: após 3 falhas seguidas) | **Adiado.** "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto." | `[09:37] Marcos` (propõe), `[09:37] Larissa` (adia) |
| **OUT-02** | **Painel/dashboard visual para o cliente acompanhar seus webhooks** | **Descartado desta fase.** "Não, agora não. Só endpoints." É projeto separado do time de frontend. | `[09:39] Marcos` (propõe), `[09:40] Larissa` (descarta) |
| **OUT-03** | **Rate limiting de saída** — limitar a vazão de chamadas disparadas contra um mesmo cliente | **Em aberto.** Decidiu-se observar e implementar apenas se virar problema real. | `[09:38] Diego` (levanta), `[09:39] Larissa` ("observar e decidir depois") |
| **OUT-04** | **Webhooks inbound** — o cliente enviando eventos para a plataforma | **Descartado por definição de escopo.** "Só saindo da gente pra eles. Eles querem receber, não mandar." | `[09:02] Sofia` (pergunta), `[09:02] Marcos` (define) |
| **OUT-05** | **Arquivamento/expurgo dos eventos já entregues** (ordem de 30 dias) | **Adiado.** Citado como necessário no futuro e colocado explicitamente fora do escopo desta feature. | `[09:08] Diego` |
| **OUT-06** | **Autorização mais restrita para o cadastro de webhooks** — hoje qualquer usuário autenticado pode manter a configuração | **Adiado.** "Por enquanto sim. Mais pra frente a gente pode endurecer." | `[09:37] Sofia` |
| **OUT-07** | **Escala horizontal do processamento** e a garantia de ordenação que ela exige | **Adiado.** Tratado como "problema do futuro"; a limitação de ordenação decorrente está registrada em RNF-07. | `[09:13] Diego`, `[09:13] Larissa` |
| **OUT-08** | **Documentação da integração no portal do desenvolvedor** | **Fora do escopo técnico.** Assumido pelo PM como trabalho paralelo. | `[09:26] Marcos`, `[09:40] Marcos` |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| **PRD-FR-01** | O cliente pode **registrar um endpoint de webhook**, informando o endereço de destino e a lista de mudanças de status que deseja receber. A credencial de assinatura é **gerada pela plataforma** e devolvida no ato do cadastro. | `[09:31] Marcos` |
| **PRD-FR-02** | O registro de webhook mantém o **endereço de destino, a credencial de assinatura, o cliente a que pertence e se está ativo**. | `[09:21] Bruno`, `[09:21] Sofia` |
| **PRD-FR-03** | O cliente pode **alterar, remover e listar** os endpoints de webhook de um cliente. | `[09:33] Bruno` |
| **PRD-FR-04** | Cada endpoint define **quais mudanças de status quer receber** — por exemplo, apenas *enviado* e *entregue*. Mudanças não assinadas por nenhum endpoint do cliente não geram evento. | `[09:33] Marcos`, `[09:34] Bruno`, `[09:34] Diego` |
| **PRD-FR-05** | O cliente pode **solicitar uma nova credencial de assinatura**. A credencial anterior permanece válida em paralelo por **24 horas**, para ele migrar seus sistemas, e é invalidada em seguida. | `[09:21] Sofia` |
| **PRD-FR-06** | **Toda mudança de status de pedido gera um evento de notificação**, registrado de forma **atômica com a própria mudança**: se o evento não puder ser registrado, a mudança de status não acontece. | `[09:40] Bruno`, `[09:41] Diego` |
| **PRD-FR-07** | A notificação entregue ao cliente descreve **a transição ocorrida** — qual pedido, de qual status para qual status, para qual cliente, quando, e os dados básicos do pedido. Não inclui a composição de itens do pedido. | `[09:43] Diego`, `[09:44] Bruno` |
| **PRD-FR-08** | Cada notificação é **assinada criptograficamente** com a credencial do endpoint de destino, de modo que o cliente possa verificar que a chamada partiu da plataforma e que o conteúdo não foi alterado no caminho. | `[09:19] Sofia`, `[09:20] Sofia` |
| **PRD-FR-09** | Cada notificação carrega um **identificador único de evento**, para que o cliente descarte duplicatas do lado dele. | `[09:25] Diego`, `[09:26] Larissa` |
| **PRD-FR-10** | Entregas que falham são **retentadas até 5 vezes**, com intervalos crescentes de **1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas**. | `[09:17] Diego`, `[09:17] Larissa` |
| **PRD-FR-11** | Esgotadas as tentativas, o evento é **preservado junto com o motivo da falha e o momento em que ocorreu**, e deixa de ser retentado automaticamente. | `[09:18] Diego` |
| **PRD-FR-12** | Um **administrador** pode **reprocessar manualmente** um evento que falhou definitivamente, recolocando-o na fila de entrega. A operação **registra quem a executou**, para auditoria. | `[09:18] Diego`, `[09:35] Diego`, `[09:36] Sofia`, `[09:36] Larissa` |
| **PRD-FR-13** | O cliente pode **consultar o histórico de entregas** de um endpoint, vendo o que foi enviado, se houve sucesso ou falha, qual foi a resposta recebida e o tempo de resposta. | `[09:34] Marcos` |
| **PRD-FR-14** | O cadastro **recusa endereços de destino sem TLS**: apenas `https` é aceito. | `[09:23] Sofia` |
| **PRD-FR-15** | Eventos cujo conteúdo ultrapasse o **limite de 64 KB não são enviados** — a plataforma trata o caso como erro em vez de truncar o conteúdo. | `[09:23] Sofia`, `[09:24] Diego`, `[09:24] Larissa` |
| **PRD-FR-16** | A manutenção da configuração de webhooks exige **usuário autenticado**; o reprocessamento de eventos falhos exige **papel de administrador**. | `[09:36] Sofia`, `[09:37] Sofia` |

## 7. Requisitos não funcionais

| ID | Requisito | Critério | Origem |
| --- | --- | --- | --- |
| **PRD-NFR-01** | **Latência de notificação** | Abaixo de 10 segundos entre a mudança de status e a entrega — a definição de "tempo real" acordada com os clientes. | `[09:02] Marcos` |
| **PRD-NFR-02** | **Não bloqueio do fluxo de pedidos** | A mudança de status não pode ser retardada nem interrompida por cliente lento ou indisponível. Um cliente fora do ar jamais pode causar a reversão de uma mudança de status. | `[09:04] Bruno` |
| **PRD-NFR-03** | **Resiliência a indisponibilidade do destino** | A janela de retentativas cobre indisponibilidades longas do cliente — a progressão adotada soma cerca de 15 horas. | `[09:15] Diego`, `[09:16] Diego`, `[09:17] Diego` |
| **PRD-NFR-04** | **Tolerância a lentidão do destino** | Chamadas que não obtêm resposta em 10 segundos são tratadas como falha e reprogramadas. | `[09:42] Diego` |
| **PRD-NFR-05** | **Garantia de entrega** | *At-least-once*: a plataforma pode entregar o mesmo evento mais de uma vez, e o cliente é responsável por descartar duplicatas pelo identificador do evento. | `[09:24] Diego`, `[09:26] Larissa` |
| **PRD-NFR-06** | **Fidelidade do conteúdo** | A notificação reflete o estado do pedido **no momento da mudança de status**, mesmo que o pedido mude novamente antes de a entrega acontecer. | `[09:52] Larissa`, `[09:52] Diego` |
| **PRD-NFR-07** | **Ordenação** *(limitação conhecida)* | A ordem de entrega é preservada **por pedido**. Não há garantia de ordenação global entre pedidos distintos, e a garantia por pedido vale enquanto o processamento não for paralelizado. Os clientes não pediram ordenação global. | `[09:12] Diego`, `[09:13] Larissa`, `[09:14] Marcos` |
| **PRD-NFR-08** | **Segurança do transporte** | Comunicação exclusivamente sobre TLS. | `[09:23] Sofia` |
| **PRD-NFR-09** | **Isolamento de credenciais** | Credencial exclusiva por endpoint, nunca compartilhada entre clientes: o vazamento de uma não compromete as demais. Deve ser rotacionável pelo próprio cliente. | `[09:21] Sofia` |
| **PRD-NFR-10** | **Limite de tamanho** | 64 KB por evento. | `[09:24] Diego`, `[09:24] Larissa` |
| **PRD-NFR-11** | **Custo de infraestrutura** | A feature não deve exigir nenhum componente de infraestrutura novo — o time é pequeno e a operação precisa continuar simples. | `[09:07] Diego` |
| **PRD-NFR-12** | **Independência operacional** | O processamento das entregas não pode ser interrompido por um reinício da API. | `[09:11] Diego` |
| **PRD-NFR-13** | **Auditabilidade** | Toda intervenção manual sobre a fila de entrega fica registrada com o seu autor. | `[09:36] Sofia` |

## 8. Decisões e trade-offs principais

> **Nota de processo.** Esta seção é o resumo, em linguagem de produto, das decisões técnicas fechadas
> na reunião. O registro formal de cada decisão — com contexto, alternativas e consequências — está nos
> [ADRs](./adrs/), e a proposta arquitetural que as conecta está no [RFC](./RFC.md).

| # | Decisão | Trade-off aceito | Registro |
| --- | --- | --- | --- |
| 1 | **Emitir o evento junto com a mudança de status, no mesmo banco de dados já existente**, em vez de chamar o cliente na hora ou introduzir uma infraestrutura de mensageria dedicada. | Ganha-se consistência absoluta (o evento existe se e somente se a mudança de status aconteceu) e nenhum custo de infraestrutura nova. Paga-se com um mecanismo de entrega próprio, que a plataforma passa a manter. | [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) |
| 2 | **Entregar de forma assíncrona, por um processo separado que verifica periodicamente se há eventos a enviar.** | Ganha-se isolamento total do fluxo de pedidos e independência de reinícios da API. Paga-se com uma latência mínima inerente ao intervalo de verificação — dentro da meta de 10 segundos com folga. | [ADR-005](./adrs/ADR-005-worker-separado-em-polling.md) |
| 3 | **Retentar 5 vezes com intervalos crescentes e, depois, parar e preservar o evento** em vez de retentar indefinidamente ou desistir cedo. | Ganha-se cobertura de indisponibilidades longas do cliente sem acumular entregas eternamente pendentes. Paga-se com a possibilidade de um evento levar até ~15 horas para ser entregue, e com a necessidade de intervenção manual no caso terminal. | [ADR-002](./adrs/ADR-002-retry-com-backoff-e-dlq.md) |
| 4 | **Assinar cada notificação com uma credencial exclusiva do endpoint**, rotacionável, em vez de uma credencial única da plataforma. | Ganha-se contenção de dano em caso de vazamento — cenário já vivido com um cliente. Paga-se com a complexidade de gerir o ciclo de vida de várias credenciais e a convivência temporária entre a antiga e a nova. | [ADR-003](./adrs/ADR-003-hmac-sha256-com-secret-por-endpoint.md) |
| 5 | **Assumir entrega *at-least-once* e transferir a deduplicação para o cliente**, em vez de perseguir entrega exatamente-uma-vez. | Ganha-se simplicidade e alinhamento com o padrão de mercado (Stripe, GitHub). Paga-se com uma exigência de integração imposta ao cliente, que precisa ser bem documentada. | [ADR-004](./adrs/ADR-004-entrega-at-least-once-com-x-event-id.md) |
| 6 | **Construir a feature reaproveitando ao máximo os padrões já existentes na plataforma** em vez de introduzir novas bibliotecas ou convenções. | Ganha-se velocidade de entrega, consistência e menor curva para o time. Paga-se com a aceitação das limitações dos padrões atuais. | [ADR-006](./adrs/ADR-006-reuso-dos-padroes-existentes.md) |

## 9. Dependências

| # | Dependência | Natureza | Origem |
| --- | --- | --- | --- |
| **DEP-01** | **Fluxo de mudança de status de pedido do OMS.** A feature se acopla ao ponto exato em que o status do pedido muda; é dele que todo evento nasce. | Técnica, interna | `[09:40] Bruno` |
| **DEP-02** | **Banco de dados MySQL existente.** Nenhuma infraestrutura adicional será provisionada. | Técnica, interna | `[09:07] Diego` |
| **DEP-03** | **Mecanismos de autenticação e autorização já existentes na plataforma**, incluindo o controle de papel administrativo. | Técnica, interna | `[09:36] Larissa` |
| **DEP-04** | **Novo processo operacional a ser publicado e monitorado** junto da API, com seu próprio ciclo de vida. | Operacional | `[09:11] Diego` |
| **DEP-05** | **Revisão de segurança pela engenheira de segurança antes do deploy**, com no mínimo **2 dias úteis** reservados, focada na assinatura e na geração de credenciais. É bloqueante para a subida. | Processo | `[09:46] Sofia`, `[09:49] Sofia` |
| **DEP-06** | **Sessão de revisão do design com os engenheiros de Pedidos e Plataforma** antes do início da implementação. | Processo | `[09:50] Larissa` |
| **DEP-07** | **Preparo do cliente.** A integração exige que o cliente exponha um endpoint sobre TLS, valide a assinatura e descarte eventos duplicados. Sem isso, a feature não entrega valor. | Externa | `[09:20] Sofia`, `[09:25] Diego`, `[09:26] Marcos` |

## 10. Riscos e mitigação

| ID | Risco | Probabilidade | Impacto | Mitigação | Origem |
| --- | --- | --- | --- | --- | --- |
| **RSK-01** | **Cliente não implementa deduplicação e processa o mesmo evento duas vezes**, gerando efeito colateral no sistema dele (ex.: baixa em duplicidade). A garantia é *at-least-once*, e a responsabilidade foi conscientemente transferida ao cliente. | Média | Alto | Enviar um identificador único por evento e documentar a exigência com destaque no portal do desenvolvedor, compromisso assumido pelo PM na reunião. | `[09:25] Sofia` (levanta o risco), `[09:25] Diego`, `[09:26] Marcos` (mitigação) |
| **RSK-02** | **Vazamento da credencial de assinatura pelo lado do cliente** — cenário já ocorrido: um cliente expôs a credencial no log da própria aplicação. | Média | Alto | Credencial exclusiva por endpoint, para conter o dano a um único destino; rotação sob demanda pelo cliente, com 24 h de convivência para migração sem perda de eventos. | `[09:22] Diego` (precedente real), `[09:21] Sofia` (mitigação) |
| **RSK-03** | **Cliente permanece indisponível além da janela de retentativas (~15 h)** e perde eventos de forma silenciosa. | Baixa | Médio | Preservar os eventos que falharam definitivamente, com o motivo, e oferecer reprocessamento manual por administrador. Marcos avaliou o risco como aceitável: "se um cliente meu cair por 15 horas, ele já tá com problema sério dele". | `[09:17] Marcos`, `[09:18] Diego` |
| **RSK-04** | **Cliente com muitos pedidos mudando de status em sequência recebe uma rajada de chamadas** e é sobrecarregado pela própria plataforma. | Média | Médio | Nenhuma mitigação nesta fase — decisão consciente de **observar e agir se virar problema**. Registrado como questão em aberto no RFC. | `[09:38] Diego`, `[09:39] Larissa` |
| **RSK-05** | **Perda da conta Atlas Comercial** caso a entrega não saia no prazo negociado. | Baixa | Alto | Escopo deliberadamente enxuto — e-mail de alerta, painel visual e rate limiting ficaram de fora justamente para caber no prazo; estimativa de 3 sprints já com a revisão de segurança incluída. | `[09:00] Marcos`, `[09:45] Marcos`, `[09:47] Larissa` |
| **RSK-06** | **Entregas fora de ordem** caso o processamento venha a ser paralelizado no futuro, quebrando a ordenação por pedido. | Baixa | Médio | Manter o processamento não paralelizado nesta fase e registrar a limitação explicitamente. Os clientes não pediram ordenação global. | `[09:12] Diego`, `[09:13] Larissa`, `[09:14] Marcos` |
| **RSK-07** | **Falha ao registrar o evento impede a mudança de status do pedido.** Como o registro é atômico com a mudança, um defeito no mecanismo de eventos propaga-se para o fluxo central de pedidos. | Baixa | Alto | Trade-off aceito conscientemente: "Não pode ter caso de status mudar e evento não sair". Mitigado por escrita puramente local ao banco — sem chamada externa no caminho crítico — e por cobertura de testes ponta a ponta prevista na estimativa. | `[09:40] Bruno`, `[09:41] Diego`, `[09:46] Larissa` |

## 11. Critérios de aceitação

A feature é considerada pronta quando:

| ID | Critério | Verificação | Origem |
| --- | --- | --- | --- |
| **AC-01** | Um cliente consegue registrar, listar, alterar e remover endpoints de webhook pela API, recebendo a credencial de assinatura no momento do cadastro. | PRD-FR-01, PRD-FR-03 | `[09:31] Marcos`, `[09:33] Bruno` |
| **AC-02** | Endereço de destino sem TLS é recusado no cadastro. | PRD-FR-14 | `[09:23] Sofia` |
| **AC-03** | Uma mudança de status assinada por um endpoint ativo resulta em notificação entregue **em menos de 10 segundos** em condições normais. | OBJ-01, PRD-NFR-01 | `[09:02] Marcos` |
| **AC-04** | Uma mudança de status **não assinada por nenhum endpoint** do cliente não gera evento algum. | PRD-FR-04 | `[09:34] Bruno` |
| **AC-05** | Se o registro do evento falhar, a mudança de status **não é efetivada** — pedido e evento são consistentes em qualquer cenário de falha. | PRD-FR-06, RSK-07 | `[09:40] Bruno`, `[09:41] Diego` |
| **AC-06** | A notificação entregue é verificável pelo cliente com a credencial do endpoint, e uma alteração no conteúdo invalida a verificação. | PRD-FR-08 | `[09:20] Sofia` |
| **AC-07** | Um endpoint indisponível recebe **exatamente 5 tentativas**, nos intervalos de 1 min, 5 min, 30 min, 2 h e 12 h, e só então o evento é dado como falho em definitivo. | PRD-FR-10, PRD-FR-11 | `[09:17] Diego`, `[09:17] Larissa` |
| **AC-08** | Um endpoint que não responde em **10 segundos** tem a chamada tratada como falha e reprogramada. | PRD-NFR-04 | `[09:42] Diego` |
| **AC-09** | Um administrador consegue reprocessar um evento que falhou definitivamente, e o reprocessamento fica registrado com o seu autor. Um usuário sem papel de administrador é impedido. | PRD-FR-12, PRD-FR-16 | `[09:35] Diego`, `[09:36] Sofia` |
| **AC-10** | O cliente consegue consultar o histórico de entregas de um endpoint, com resultado, conteúdo enviado, resposta recebida e tempo de resposta. | PRD-FR-13 | `[09:34] Marcos` |
| **AC-11** | A rotação de credencial mantém a anterior válida por **24 horas** e a invalida depois disso. | PRD-FR-05 | `[09:21] Sofia` |
| **AC-12** | Um evento acima de **64 KB** não é enviado e é tratado como erro. | PRD-FR-15, PRD-NFR-10 | `[09:24] Diego`, `[09:24] Larissa` |
| **AC-13** | A notificação reflete o estado do pedido no momento da mudança de status, ainda que o pedido tenha mudado novamente antes da entrega. | PRD-NFR-06 | `[09:52] Larissa` |
| **AC-14** | Um reinício da API **não interrompe** a entrega de eventos pendentes. | PRD-NFR-12 | `[09:11] Diego` |
| **AC-15** | A revisão de segurança foi concluída antes do deploy, com no mínimo 2 dias úteis dedicados. | DEP-05 | `[09:46] Sofia`, `[09:49] Sofia` |

## 12. Estratégia de testes e validação

A estimativa acordada reserva explicitamente meia sprint para "integração no fluxo de pedidos e testes
ponta a ponta" (`[09:46] Larissa`), e dois dias úteis de revisão de segurança antes do deploy
(`[09:46] Sofia`). A estratégia abaixo detalha o que essa validação precisa cobrir; o detalhamento
técnico dos testes está no [FDD](./FDD.md).

| Nível | O que valida | Cobertura mínima esperada |
| --- | --- | --- |
| **Testes automatizados de unidade** | A lógica isolada de decisão: quais transições geram evento, qual endpoint assina qual mudança, cálculo do próximo horário de retentativa, geração e verificação da assinatura, aplicação do limite de tamanho. | AC-04, AC-06, AC-07, AC-12 |
| **Testes automatizados de integração** | O comportamento transacional: mudança de status e registro do evento vivem ou morrem juntos, inclusive nos caminhos de falha. É o teste que protege RSK-07. | AC-05 |
| **Testes automatizados de API** | Os fluxos completos de cadastro, alteração, remoção, listagem, rotação de credencial, consulta de entregas e reprocessamento, incluindo os casos de recusa (destino sem TLS, usuário sem papel de administrador). | AC-01, AC-02, AC-09, AC-10, AC-11 |
| **Validação de comportamento sob falha** | O ciclo de retentativas contra um destino indisponível e contra um destino lento, até o esgotamento das tentativas e a preservação do evento falho. | AC-07, AC-08 |
| **Validação ponta a ponta** | Uma mudança de status real produzindo uma notificação verificável no destino dentro da janela de 10 segundos, e a continuidade da entrega após um reinício da API. | AC-03, AC-13, AC-14 |
| **Revisão de segurança** | Revisão manual e bloqueante conduzida pela engenheira de segurança, com foco declarado na assinatura e na geração de credenciais: *"HMAC e geração de secret eu quero olhar com calma"*. | AC-06, AC-11, AC-15 |

### Validação com o cliente

Os três clientes que originaram a demanda (`[09:00] Marcos`) são os validadores naturais da entrega. O
critério de sucesso do ponto de vista deles é o declarado na reunião: parar de consultar a API em busca
de mudanças e ser avisado em menos de 10 segundos, sem intervenção manual (`[09:02] Marcos`). A
comunicação e o acompanhamento junto a eles são conduzidos pelo PM, que assumiu tanto a atualização dos
clientes (`[09:49] Marcos`) quanto a documentação da integração no portal (`[09:40] Marcos`).
