# ADR-004 — Entrega at-least-once com deduplicação por `X-Event-Id`

| | |
| --- | --- |
| **Status** | Aceita |
| **Data** | Reunião de definição da feature (`TRANSCRICAO.md`, quinta-feira, 09:00–09:53) |
| **Decisores** | Diego (Eng. Sênior, Plataforma), Larissa (Tech Lead), Sofia (Eng. de Segurança), Marcos (PM) |
| **RFC** | [RFC — Entrega de Notificações de Pedido por Webhook](../RFC.md), §3.2e |
| **Relacionadas** | [ADR-002](./ADR-002-retry-com-backoff-e-dlq.md) (as retentativas são a principal fonte de duplicidade) · [ADR-001](./ADR-001-outbox-no-mysql.md) (onde o identificador é gerado) |

## Contexto

Qualquer sistema de entrega com retentativa produz duplicidade. O caso é elementar: o worker envia, o
cliente processa, mas a resposta se perde no caminho — para nós é falha, e a entrega é reprogramada; para
o cliente, o evento chegou duas vezes.

Diego trouxe o ponto explicitamente: "a gente vai garantir at-least-once. Pode acontecer de o cliente
receber o mesmo evento duas vezes. Ele tem que estar preparado" (`[09:24] Diego`). Bruno respondeu com a
pergunta prática: "E como ele diferencia?" (`[09:25] Bruno`).

Garantias mais fortes existem, mas custam. Sofia nomeou o preço da escolha antes de ela ser fechada:
"Isso joga responsabilidade pro cliente" (`[09:25] Sofia`) — a observação está registrada aqui porque é o
trade-off, não uma objeção vencida.

## Decisão

**Assumir entrega at-least-once e transferir a deduplicação ao cliente, por meio de um identificador
único de evento enviado no header `X-Event-Id`.**

- O identificador é um **UUID gerado no momento em que o evento entra na outbox**, e é **único por
  evento** — não por tentativa (`[09:25] Diego`). Todas as retentativas de um mesmo evento carregam o
  mesmo `X-Event-Id`, que é exatamente o que torna a deduplicação possível.
- O cliente **dedupica pelo `event_id` do lado dele** (`[09:25] Diego`).
- O `X-Event-Id` acompanha os demais headers de cada envio, junto com a assinatura, o timestamp do envio e
  o identificador do endpoint (`[09:44] Diego`, `[09:44] Sofia`).
- A exigência é **documentada com destaque** para os clientes: "Eu posso documentar isso bem destacado no
  portal de desenvolvedor pros clientes, sem problema" (`[09:26] Marcos`).

Fechada em `[09:26] Larissa`: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."

## Alternativas Consideradas

### A. Entrega exactly-once

Garantir que cada evento chegue exatamente uma vez, sem exigir nada do cliente.

**Rejeitada** por complexidade desproporcional. "Garantir exactly-once exigiria coordenação dos dois
lados e fica muito mais complexo. At-least-once com event_id resolve 99% dos casos" (`[09:25] Diego`).
Coordenação "dos dois lados" é o cerne: exactly-once não é algo que a plataforma possa entregar
sozinha — exigiria um protocolo de confirmação com o cliente, ou seja, mais trabalho de integração para
ele do que a deduplicação que estamos pedindo.

Além disso, a alternativa foi pesada contra o mercado: "é o padrão de mercado. Stripe faz assim, GitHub
faz assim" (`[09:25] Diego`). Clientes que já integram webhooks de outros fornecedores tendem a ter a
deduplicação implementada.

### B. At-most-once — não retentar

Enviar uma vez e desistir na falha, eliminando a duplicidade pela raiz.

**Rejeitada por incompatibilidade com [ADR-002](./ADR-002-retry-com-backoff-e-dlq.md).** Trocaria
duplicidade por perda de evento, e a resiliência a indisponibilidade do cliente é requisito central da
feature (`[09:14] Larissa`, `[09:15] Diego`). Entre entregar demais e entregar de menos, a reunião optou
sem hesitação pelo primeiro.

## Consequências

### Positivas

- **Desenho de entrega simples.** O worker precisa apenas tentar até obter sucesso; não há protocolo de
  confirmação, estado compartilhado nem coordenação com o cliente.
- **Composição limpa com a política de retentativa.** [ADR-002](./ADR-002-retry-com-backoff-e-dlq.md)
  pode ser tão agressivo quanto necessário sem introduzir risco de duplicidade *nova* — o identificador
  estável absorve todas as tentativas.
- **Familiaridade de mercado.** Clientes que já consomem webhooks de Stripe ou GitHub encontram a mesma
  semântica e reaproveitam a implementação que já têm.
- **Correção sob falha parcial.** Resposta perdida, timeout e queda no meio do processamento resultam em
  reenvio, nunca em evento perdido.

### Negativas

- **Responsabilidade transferida ao cliente** — o custo nomeado por Sofia (`[09:25] Sofia`). Parte do
  esforço de integração passa a ser dele.
- **A correção final está fora do nosso controle.** Se o cliente não deduplicar, ele processa o evento
  duas vezes, com efeito colateral no sistema dele — baixa em duplicidade, e-mail repetido ao consumidor
  final. Não temos como verificar nem impor a implementação correta.
- **Dependência de documentação.** A mitigação é essencialmente documental, e depende do compromisso do
  PM com o portal do desenvolvedor (`[09:26] Marcos`). Documentação fraca aqui vira incidente do cliente.
- **Suporte a incidentes que não são nossos.** Duplicidade não tratada tende a chegar como reclamação
  contra a plataforma, mesmo sendo comportamento contratado e documentado.
