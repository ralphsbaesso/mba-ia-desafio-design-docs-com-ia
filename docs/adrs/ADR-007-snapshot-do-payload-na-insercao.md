# ADR-007 — Snapshot do payload no momento da inserção na outbox

| | |
| --- | --- |
| **Status** | Aceita |
| **Data** | Reunião de definição da feature (`TRANSCRICAO.md`, quinta-feira, 09:00–09:53) |
| **Decisores** | Larissa (Tech Lead), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma) |
| **RFC** | [RFC — Entrega de Notificações de Pedido por Webhook](../RFC.md), §3.3 |
| **Relacionadas** | [ADR-001](./ADR-001-outbox-no-mysql.md) (a tabela onde o snapshot é gravado) · [ADR-002](./ADR-002-retry-com-backoff-e-dlq.md) (o retry é o que torna o intervalo relevante) |

## Contexto

Fechado o padrão outbox, ficou uma questão de modelagem que Bruno levantou no fim da reunião: "o evento
da outbox guarda o payload renderizado já, ou guarda só order_id e renderiza na hora do envio?"
(`[09:51] Bruno`).

A pergunta parece pequena e não é. Existe **intervalo de tempo** entre a mudança de status e a entrega —
no mínimo o ciclo de polling de 2 segundos ([ADR-005](./ADR-005-worker-separado-em-polling.md)), e até
cerca de 15 horas quando o retry entra em ação ([ADR-002](./ADR-002-retry-com-backoff-e-dlq.md)). Nesse
intervalo o pedido pode mudar de novo: a máquina de estados admite sequências rápidas como PAID →
PROCESSING → SHIPPED (`src/modules/orders/order.status.ts`), e Larissa já havia usado exatamente esse
cenário ao discutir ordenação (`[09:12] Larissa`).

A escolha define, portanto, **o que o cliente recebe quando o pedido muda entre a transição e a
entrega**.

## Decisão

**O evento guarda o payload já renderizado no momento da inserção — um snapshot do estado do pedido
quando a transição ocorreu.**

Larissa decidiu com a justificativa: "Eu prefiro renderizado já, na hora da inserção. Se o pedido mudar
depois, o evento ainda reflete o estado de quando o status mudou. Senão tem caso esquisito"
(`[09:52] Larissa`). Diego concordou — "Concordo, snapshot na inserção" (`[09:52] Diego`) — e Bruno
fechou: "Beleza, snapshot. Decidido" (`[09:52] Bruno`).

O conteúdo do snapshot é o payload enxuto definido em `[09:43] Diego`: identificador do evento, tipo do
evento, timestamp, identificação do pedido, status de origem e destino, cliente e os campos básicos do
pedido — sem a composição de itens. A composição exata está no [FDD](../FDD.md).

## Alternativas Consideradas

### A. Guardar apenas a referência ao pedido e renderizar no envio

A outbox guardaria o identificador do pedido e os status de origem e destino; o worker leria o pedido no
banco e montaria o payload na hora de enviar.

**Rejeitada** por quebrar a fidelidade do evento ao momento que ele descreve. O cliente receberia uma
notificação anunciando uma transição passada, mas carregando os dados atuais do pedido — o "caso
esquisito" a que Larissa se referiu (`[09:52] Larissa`). Numa retentativa 12 horas depois, a divergência
pode ser grande, e o cliente não teria como perceber a inconsistência.

Há um efeito colateral menos óbvio: renderizar no envio exigiria uma leitura adicional do pedido a cada
tentativa, multiplicando consultas ao banco pelo número de retentativas.

A alternativa não é sem mérito — economizaria espaço e permitiria corrigir o formato do payload
retroativamente para eventos ainda não entregues. Nenhum dos dois ganhos foi considerado suficiente
diante da perda de fidelidade.

## Consequências

### Positivas

- **O evento é fiel ao momento que descreve**, qualquer que seja o atraso até a entrega — inclusive nas
  retentativas de 12 horas.
- **Entrega independente do estado atual do banco.** O worker não precisa reler o pedido; se o pedido for
  alterado, ou mesmo removido, o evento pendente continua íntegro e enviável.
- **Menos leituras por tentativa.** Cada retentativa reenvia o mesmo conteúdo já materializado, sem
  consultar o pedido novamente.
- **Evidência de auditoria naturalmente preservada.** O que foi enviado fica registrado exatamente como
  foi enviado, o que também serve à dead letter e ao histórico de entregas.
- **Assinatura estável.** Como o corpo não muda entre tentativas, a assinatura HMAC
  ([ADR-003](./ADR-003-hmac-sha256-com-secret-por-endpoint.md)) permanece consistente para o mesmo
  evento.

### Negativas

- **Duplicação de dados.** Os campos do pedido passam a existir também na outbox, aumentando o volume da
  tabela — que já cresce sem política de retenção
  ([ADR-001](./ADR-001-outbox-no-mysql.md), consequências negativas).
- **O formato do payload congela na inserção.** Uma mudança no formato só vale para eventos criados
  depois dela; eventos pendentes serão entregues no formato antigo, e uma correção de bug no payload não
  alcança o que já está na fila.
- **O snapshot pode chegar "velho" ao cliente.** Numa entrega tardia, o cliente recebe dados que não
  refletem o estado atual do pedido. É o comportamento pretendido, mas precisa estar documentado — do
  contrário vira dúvida de integração. O caminho para o dado atual é o próprio `GET` do pedido
  (`[09:43] Diego`).
- **Reforça o limite de tamanho.** Materializar o payload é o que torna o teto de 64 KB
  (`[09:24] Diego`) uma verificação sobre uma linha de tabela, e não apenas sobre a requisição de saída.
