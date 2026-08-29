# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que este repositório é

Este repo é a entrega de um **desafio puramente documental** (`the-challenge/INDEX.md`): transformar a transcrição de uma reunião técnica (`TRANSCRICAO.md`) + o código existente em um pacote de design docs para a feature **Sistema de Webhooks de Notificação de Pedidos**.

A aplicação em `src/` é um Order Management System (Node 20 + TypeScript + Express + Prisma/MySQL) **funcional e congelado**: ela existe como *contexto e referência*, e não tem hoje nenhum mecanismo de eventos, filas ou notificação externa — o vácuo que a feature documentada vai preencher.

## Restrição absoluta

**Nunca edite código.** `src/`, `prisma/`, `tests/`, `docker-compose.yml`, `package.json`, `tsconfig*`, `.eslintrc.json`, `vitest.config.ts`, `TRANSCRICAO.md` e `the-challenge/` são somente leitura.

Os únicos arquivos graváveis da entrega são:

| Arquivo | Estado atual |
| --- | --- |
| `README.md` | placeholder ("Em construção") — vira o README do *processo* |
| `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md` | esqueletos com comentário `<!-- ... -->` |
| `docs/adrs/ADR-NNN-titulo-em-kebab-case.md` | 5 a 8 arquivos a criar (só existe o `README.md` da pasta) |

## Regra de ouro: rastreabilidade

Toda afirmação nos documentos precisa ter origem em `TRANSCRICAO.md` ou no código. **Não inventar** requisitos, decisões, números, restrições, métricas ou nomes de arquivo.

Teste prático: se não dá pra preencher a coluna **Localização** do `docs/TRACKER.md` para uma linha, ela é alucinação — corrija ou remova.

- Fonte `TRANSCRICAO` → localização no formato `[hh:mm] Nome` (a transcrição vai de `[09:00]` a `[09:53]`).
- Fonte `CODIGO` → caminho real. **Confirme com `grep`/`ls` antes de citar qualquer path, símbolo ou linha.**

Gates quantitativos do enunciado que costumam ser esquecidos: ≥8 requisitos funcionais no PRD; ≥2 itens em "Fora de escopo" e ≥2 riscos com probabilidade/impacto/mitigação; ≥2 alternativas descartadas e ≥2 questões em aberto no RFC, com links pra ≥2 ADRs; ≥4 endpoints com request/response e status codes no FDD; ≥4 caminhos reais na seção "Integração com o sistema existente"; ≥70% das linhas do tracker com fonte `TRANSCRICAO` e ≥5 com `CODIGO`.

## Altura de cada documento (não duplicar)

Conteúdo repetido entre documentos é sinal de que está no lugar errado.

- **PRD** — produto/negócio: por que e o quê (problema, escopo, métricas).
- **RFC** — arquitetura, 2 a 4 páginas: o que propomos, quais alternativas caíram e o que ficou em aberto. Metadados usam os participantes da reunião como revisores. **Não desce ao detalhe do FDD.**
- **ADRs** — uma decisão fechada por arquivo, formato MADR (Status, Contexto, Decisão, Alternativas Consideradas, Consequências com trade-off explícito).
- **FDD** — implementação: fluxos, contratos, matriz de erros `WEBHOOK_*`, resiliência, observabilidade.
- **TRACKER** — referência cruzada transversal.

Ordem de produção sugerida: **ADRs → RFC → FDD → PRD → Tracker → README**.

## Trabalhando com a transcrição

Formato: `[hh:mm] Nome: fala`. Participantes: Larissa (Tech Lead, conduz), Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Eng. de Segurança).

A reunião mistura decisões fechadas, requisitos, ideias **descartadas** e itens **adiados para fases futuras**. Separar o que entra do que não entra é parte central do desafio — itens descartados que aparecem como requisito são erro de entrega, e itens adiados pertencem a "Fora de escopo"/"Questões em aberto", não aos requisitos. Use prompts dirigidos por recorte (ex.: "liste apenas o que foi explicitamente descartado, com timestamp") em vez de pedidos genéricos.

As 6 decisões principais a cobrir (mínimo 5) estão listadas em `the-challenge/INDEX.md`: outbox no MySQL, retry com backoff + DLQ, HMAC-SHA256 com secret por endpoint, at-least-once com `X-Event-Id`, worker separado em polling, e reuso dos padrões existentes.

## Mapa do código (base da seção "Integração com o sistema existente")

Âncoras reais, verificadas, que o FDD e ao menos um ADR precisam referenciar:

- `src/modules/orders/order.service.ts:126` — `changeStatus()` roda tudo dentro de `prisma.$transaction`: valida transição, debita/repõe estoque, `order.update` + `orderStatusHistory.create`. É o ponto natural de escrita do evento na outbox (mesma transação).
- `src/modules/orders/order.status.ts` — máquina de estados (`canTransition`, `allowedTransitions`, `isTerminal`, `shouldDebitStock`, `shouldReplenishStock`); define quais transições existem e podem gerar evento.
- `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` — `AppError(message, statusCode, errorCode, details)` e subclasses (`NotFoundError`, `ConflictError`, `InvalidStatusTransitionError`, …). Os códigos `WEBHOOK_*` devem seguir esse padrão em vez de criar um novo.
- `src/middlewares/error.middleware.ts:14` — error handler central; formato de resposta `{ error: { code, message, details? } }`, com tratamento de `ZodError` e `PrismaClientKnownRequestError` (P2002/P2025).
- `src/middlewares/auth.middleware.ts` — `authenticate` (JWT Bearer) e `requireRole(...)` (linha 49); hoje `requireRole('ADMIN')` só é usado em `src/modules/users/user.routes.ts:15`.
- `src/middlewares/validate.middleware.ts` — `validate({ body, query, params })` com Zod, aplicado por rota.
- `src/shared/logger/index.ts` — Pino com `redact` de campos sensíveis (`*.token`, `*.password`, …) — relevante pro tratamento do secret HMAC.
- `src/middlewares/request-logger.middleware.ts` — `X-Request-Id` (entrada ou uuid) + log `http_request` com duração.
- `src/routes/index.ts` — `buildApiRouter()`; é onde um router `/webhooks` se pluga.
- `src/app.ts` — `buildControllers()`/`buildApp()`: DI manual, repository → service → controller.
- `prisma/schema.prisma` — convenções: `@id @default(uuid()) @db.Char(36)`, `@@map` snake_case, valores em centavos (`Int`), `OrderStatusHistory` como precedente de tabela de auditoria.
- Padrão de módulo (replicar num módulo de webhooks): `<nome>.routes.ts` / `.controller.ts` / `.service.ts` / `.repository.ts` / `.schemas.ts` em `src/modules/<nome>/`.
- `src/shared/http/response.ts` — `paginated()`, formato de listagem para endpoints de consulta (entregas, DLQ).
- `tests/` — Vitest + Supertest, com `tests/helpers/factories.ts`.

## Comandos

Só para *verificar* afirmações sobre o código — a entrega não inclui rodar ou alterar a aplicação.

```bash
docker compose up -d          # MySQL
npm install
npm test                      # vitest run
npm test -- tests/orders.test.ts        # um arquivo
npm test -- -t "nome do teste"          # um teste
npm run lint
npm run db:migrate            # prisma migrate dev
npm run db:seed
npm run dev                   # tsx watch src/server.ts
```
