# ADR-002 — Retentativa com backoff exponencial e Dead Letter Queue

| | |
| --- | --- |
| **Status** | Aceita |
| **Data** | Reunião de definição da feature (`TRANSCRICAO.md`, quinta-feira, 09:00–09:53) |
| **Decisores** | Larissa (Tech Lead), Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Marcos (PM), Sofia (Eng. de Segurança) |
| **RFC** | [RFC — Entrega de Notificações de Pedido por Webhook](../RFC.md), §3.2c |
| **Relacionadas** | [ADR-001](./ADR-001-outbox-no-mysql.md) (de onde vem o evento) · [ADR-005](./ADR-005-worker-separado-em-polling.md) (quem executa as tentativas) |

## Contexto

O endpoint de destino está fora do nosso controle: pode estar fora do ar, lento, ou responder erro. A
pergunta foi posta diretamente: "Se o cliente tá offline, o que a gente faz?" (`[09:14] Larissa`).

Três fatos delimitam a resposta:

- **Indisponibilidades reais não são curtas.** Diego citou um caso concreto: "Já tinha cliente nosso com
  indisponibilidade de duas horas em manutenção planejada" (`[09:16] Diego`).
- **Um evento não pode ficar pendurado para sempre.** Retentar indefinidamente cria "o problema de evento
  ficar pendurado pra sempre se o cliente sumiu" (`[09:15] Diego`).
- **Existe tolerância de negócio para a janela.** Marcos avaliou: "Se um cliente meu cair por 15 horas,
  ele já tá com problema sério dele. Acho aceitável" (`[09:17] Marcos`).

Falta ainda decidir o que acontece com o evento que esgota as tentativas: some, fica marcado na própria
outbox, ou vai para outro lugar (`[09:17] Larissa`).

## Decisão

**Backoff exponencial com teto de 5 tentativas, seguido de Dead Letter Queue em tabela separada, com
reprocessamento manual por administrador.**

- **5 tentativas**, com a progressão **1 minuto → 5 minutos → 30 minutos → 2 horas → 12 horas**
  (`[09:17] Diego`). O total soma quase 15 horas entre a primeira falha e a última tentativa. Fechado em
  `[09:17] Larissa`: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h".
- **Uma chamada que não responde em 10 segundos conta como falha** e é reprogramada (`[09:42] Diego`).
- Esgotadas as tentativas, o evento é movido para uma **tabela de dead letter separada**, guardando o
  payload, o motivo da falha e o momento — "Mais limpa a leitura da outbox principal, e fica como
  evidence pra debug e reprocessamento" (`[09:18] Diego`).
- O reprocessamento é **manual, por endpoint administrativo**, recolocando o evento na outbox como
  pendente (`[09:18] Diego`, `[09:35] Diego`).
- O replay **exige papel de administrador** e **registra quem o executou**, para auditoria: "Mexer em
  fila de entrega de notificação não é coisa de operador. E o endpoint de admin tem que logar quem fez o
  replay" (`[09:36] Sofia`). O guard reaproveita o `requireRole` já existente
  (`src/middlewares/auth.middleware.ts:49`), decisão explícita em `[09:36] Larissa`.

## Alternativas Consideradas

### A. Três tentativas, com progressão mais agressiva

Proposta por Bruno: "3 não é melhor? Mais agressivo" (`[09:16] Bruno`).

**Rejeitada.** Cobriria uma janela pequena demais: "Se o cliente teve indisponibilidade de manhã, a gente
retentaria três vezes em 30 minutos e mataria" (`[09:16] Diego`). O caso real dos 2 h de manutenção
planejada cairia fora dessa janela.

### B. Retentativa indefinida com backoff

Sem teto de tentativas, espaçando cada vez mais.

**Rejeitada.** Diego apontou o defeito antes mesmo de a alternativa ser defendida: traz "o problema de
evento ficar pendurado pra sempre se o cliente sumiu" (`[09:15] Diego`). Sem um estado terminal, não há
como distinguir cliente temporariamente instável de cliente que desapareceu, e a outbox nunca esvazia.

### C. Marcar o evento como `failed` na própria outbox, sem tabela separada

Alternativa colocada por Larissa ao abrir o ponto: "Faz numa tabela separada ou marca como 'failed' na
própria outbox?" (`[09:17] Larissa`).

**Rejeitada** por legibilidade operacional. A tabela separada "mais limpa a leitura da outbox principal, e
fica como evidence pra debug e reprocessamento" (`[09:18] Diego`) — a outbox permanece uma fila de
trabalho, e o cemitério não polui suas consultas nem seus índices.

## Consequências

### Positivas

- **Cobre indisponibilidades longas e reais.** A janela de ~15 h absorve tanto quedas curtas quanto
  manutenções planejadas de horas, o cenário concreto que motivou o número.
- **Termina.** Um evento tem estado final; a outbox não acumula entregas eternamente pendentes.
- **Nenhum evento é perdido silenciosamente.** O que falha em definitivo fica registrado com o motivo,
  disponível para investigação e reenvio.
- **O backoff protege o cliente.** O espaçamento crescente evita martelar um destino que já está em
  dificuldade.
- **Operação auditável.** Toda intervenção manual na fila fica atribuída a um administrador identificado.

### Negativas

- **Latência de cauda alta.** Um evento pode levar até cerca de 15 horas para ser entregue — muito além
  da meta de 10 segundos, que vale para o caminho feliz. Aceito explicitamente por Marcos
  (`[09:17] Marcos`).
- **Recuperação exige gente.** Não há reprocessamento automático da dead letter; alguém precisa perceber
  o acúmulo e agir. Como não há alerta ativo nesta fase — o e-mail de aviso ficou fora de escopo
  (`[09:37] Larissa`) — a descoberta depende de o cliente consultar o histórico de entregas ou reclamar.
- **Uma tabela a mais para modelar, operar e reter.** A dead letter herda o mesmo problema de crescimento
  sem política de expurgo.
- **Números fixos.** A progressão e o teto foram escolhidos com base em um caso real, não em medição de
  produção. Reavaliá-los exige dados que ainda não temos.
