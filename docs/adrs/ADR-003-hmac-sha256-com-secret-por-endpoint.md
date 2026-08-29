# ADR-003 — Assinatura HMAC-SHA256 com secret exclusiva por endpoint

| | |
| --- | --- |
| **Status** | Aceita — sujeita a revisão de segurança bloqueante antes do deploy |
| **Data** | Reunião de definição da feature (`TRANSCRICAO.md`, quinta-feira, 09:00–09:53) |
| **Decisores** | Sofia (Eng. de Segurança, condutora do bloco), Larissa (Tech Lead), Bruno (Eng. Pleno), Diego (Eng. Sênior) |
| **RFC** | [RFC — Entrega de Notificações de Pedido por Webhook](../RFC.md), §3.2d |
| **Relacionadas** | [ADR-006](./ADR-006-reuso-dos-padroes-existentes.md) (validação Zod e tratamento de dados sensíveis em log) |

## Contexto

Sofia enquadrou o problema ao abrir o bloco de segurança: "a gente tá expondo eventos com dados de
pedidos pra um endpoint fora da nossa infra. O cliente tem que conseguir validar que a requisição veio
realmente da gente, e que ninguém adulterou o payload no meio" (`[09:19] Sofia`).

São duas exigências distintas — **autenticidade** (a chamada partiu da plataforma) e **integridade** (o
conteúdo não foi alterado em trânsito) — e ambas precisam ser verificáveis **pelo cliente**, sozinho, sem
consultar a plataforma.

Há também um precedente concreto de vazamento no ecossistema de clientes: "A gente já teve cliente que
vazou secret em log de aplicação dele uma vez" (`[09:22] Diego`). O desenho precisa presumir que uma
credencial **vai** vazar em algum momento, e limitar o dano quando isso acontecer.

## Decisão

**HMAC-SHA256 sobre o corpo da requisição, com secret exclusiva por endpoint de webhook e suporte a
rotação com período de convivência de 24 horas.**

- **Algoritmo: HMAC-SHA256.** "HMAC-SHA256 é o padrão de mercado, todo cliente sério tem biblioteca pra
  isso" (`[09:20] Sofia`). A assinatura vai num header dedicado, e o cliente a recalcula do lado dele
  (`[09:20] Sofia`).
- **Escopo do segredo: por endpoint, não por plataforma.** "Cada endpoint de webhook do cliente tem que
  ter uma secret única. Não é uma secret global da nossa plataforma. Senão se vaza uma, vaza tudo"
  (`[09:21] Sofia`).
- **A secret é gerada pela plataforma** e devolvida ao cliente no ato do cadastro (`[09:31] Marcos`); o
  registro do webhook guarda url, secret, cliente e estado ativo (`[09:21] Bruno`, `[09:21] Sofia`).
- **Rotação sob demanda, com grace period de 24 h.** "Endpoint pro cliente conseguir pedir nova secret
  pela API. Quando ele rotaciona, a antiga fica válida por 24 horas em paralelo, pra ele ter tempo de
  migrar os sistemas dele. Depois disso, a antiga morre" (`[09:21] Sofia`).
- **TLS obrigatório no destino.** URL precisa ser `https`; `http` é recusado com erro de validação. Sofia
  qualificou o próprio ponto: "isso na verdade nem é decisão arquitetural, é só uma validação no schema
  Zod" (`[09:23] Sofia`) — implementável com o `validate` já existente
  (`src/middlewares/validate.middleware.ts:11`).

Consolidado em `[09:22] Sofia`: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint,
suporte a rotação com grace period de 24h".

A decisão é **condicionada a uma revisão de segurança bloqueante**: Sofia reservou no mínimo dois dias
úteis antes do deploy, com foco declarado neste ADR — "HMAC e geração de secret eu quero olhar com calma"
(`[09:46] Sofia`), reforçado no fechamento da call (`[09:49] Sofia`).

## Alternativas Consideradas

### A. Secret global única da plataforma

Uma credencial compartilhada, usada para assinar todos os envios a todos os clientes.

**Rejeitada** pelo raio de dano. "Senão se vaza uma, vaza tudo" (`[09:21] Sofia`). Com o precedente real
de um cliente que expôs a credencial no próprio log (`[09:22] Diego`), a probabilidade de vazamento é
tratada como certeza eventual — e uma credencial global transformaria um incidente isolado de um cliente
em comprometimento de toda a base.

### B. Confiar apenas no TLS, sem assinatura de payload

O transporte já é cifrado e autenticado; poderia bastar.

**Rejeitada implicitamente pelo enquadramento de Sofia** (`[09:19] Sofia`): TLS protege o canal, mas não
dá ao cliente meio de provar **quem** originou a requisição que ele recebeu. Sem assinatura, qualquer
parte capaz de alcançar o endpoint dele pode se passar pela plataforma. TLS e assinatura resolvem
problemas diferentes e ambos foram exigidos (`[09:23] Sofia`).

### C. Rotação imediata, sem período de convivência

Trocar a secret e invalidar a anterior no mesmo instante.

**Rejeitada** por indisponibilidade induzida no cliente. Entre a rotação e a atualização do sistema do
cliente, todo evento entregue falharia na verificação — a plataforma provocaria o incidente que a rotação
pretende evitar. As 24 h de convivência existem "pra ele ter tempo de migrar os sistemas dele"
(`[09:21] Sofia`).

## Consequências

### Positivas

- **Verificação autônoma pelo cliente**, com um algoritmo padrão de mercado para o qual toda linguagem
  relevante tem biblioteca — barreira de integração baixa.
- **Dano contido por construção.** Um vazamento compromete um único endpoint, não a base de clientes.
- **Rotação sem janela de indisponibilidade.** O cliente migra no seu próprio ritmo dentro das 24 h, sem
  perder eventos.
- **Resposta operacional a incidente.** Diante de suspeita de vazamento, o próprio cliente rotaciona pela
  API, sem abrir chamado nem depender do nosso time.

### Negativas

- **Ciclo de vida de credenciais a gerir.** Geração, armazenamento, exibição única no cadastro, rotação e
  expiração da secret antiga — cada etapa é uma superfície de ataque nova, e é justamente o que a revisão
  de segurança bloqueante vai examinar.
- **Duas secrets válidas simultaneamente durante 24 h.** A janela de convivência é, por definição, uma
  janela em que uma credencial possivelmente já vazada continua aceita. É o preço consciente de não
  derrubar a integração do cliente.
- **A verificação depende do cliente.** Se ele não conferir a assinatura, ela não protege nada. A
  plataforma não tem como impor nem auditar isso.
- **Risco de exposição da secret nos nossos próprios logs.** O logger do projeto redige campos sensíveis
  (`src/shared/logger/index.ts`, lista `redact` nas linhas 4–11), mas hoje cobre `*.token`, `*.password`,
  `*.passwordHash` e `*.accessToken` — **não** `*.secret`. Estender essa lista é requisito derivado desta
  decisão, detalhado no [FDD](../FDD.md).
- **Custo de suporte na integração.** Assinatura incorreta é uma das causas mais comuns de atrito na
  adoção de webhooks, e a documentação precisa ser explícita quanto ao que exatamente é assinado.
