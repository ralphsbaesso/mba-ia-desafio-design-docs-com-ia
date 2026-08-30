# Da Reunião ao Documento — Sistema de Webhooks de Notificação de Pedidos

Entrega do desafio [**Design Docs Gerados por IA**](./the-challenge/INDEX.md): transformar a transcrição
de uma reunião técnica de 55 minutos e o código de um Order Management System em produção num pacote de
design docs acionável.

Este README documenta **o processo de produção**. Os documentos entregues estão em [`docs/`](./docs/).

---

## Sobre o desafio

Uma empresa que opera um Order Management System vai construir um **Sistema de Webhooks de Notificação
de Pedidos**. Cinco pessoas — tech lead, PM, dois engenheiros e uma engenheira de segurança — fecharam a
decisão técnica numa call, e **nada foi registrado além da transcrição** (`TRANSCRICAO.md`). A tarefa é
produzir, a partir dela e do código existente, a documentação que permite ao time começar a implementar:
PRD, RFC, ADRs, FDD, um tracker de rastreabilidade e este README.

A dificuldade real do desafio não é escrever muito, é **filtrar**. A reunião mistura decisões fechadas,
ideias explicitamente descartadas, itens adiados para fases futuras e números que são apenas ilustração.
Um item descartado que reaparece como requisito é erro de entrega. Por isso a regra que governa todo o
pacote é uma só: **toda afirmação precisa ter origem rastreável na transcrição (`[hh:mm] Nome`) ou num
caminho real do código** — e o `docs/TRACKER.md` existe justamente para tornar essa regra verificável,
linha a linha.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
| --- | --- |
| **Claude Code** (Claude Opus 5, CLI no terminal) | Ferramenta única de produção. Com acesso direto ao repositório, foi usada para ler a transcrição, mapear o código, redigir os documentos e — o mais importante — **rodar as verificações mecânicas** de cada fase: `grep`/`sed` para confirmar cada `arquivo:linha` citado e scripts Python para validar timestamps, links e os gates quantitativos do tracker. |
| **`CLAUDE.md`** (memória de projeto) | Arquivo de instruções lido automaticamente a cada sessão. Consolida as restrições do enunciado, os gates quantitativos, a altura de cada documento e o mapa de âncoras do código. Evita ter que recontextualizar a IA a cada interação. |
| **Skill `git-commit-generator`** | Skill customizada para gerar as mensagens de commit no padrão Conventional Commits, uma por fase. |

O acesso direto ao repositório foi decisivo: em vez de *afirmar* que `changeStatus` roda dentro de uma
transação, foi possível **abrir o arquivo e confirmar** — e, mais de uma vez, descobrir que a afirmação
estava errada (ver *Iterações*).

## Workflow adotado

O trabalho foi organizado em **8 fases, cada uma terminando num gate de saída verificável**. Nenhuma
fase avança com gate aberto.

```
Fase 0  Enquadramento        → CLAUDE.md (restrições, gates, altura de cada documento)
Fase 1  Insumos              → 6 artefatos de triagem em docs/notas/
Fase 2  PRD                  → docs/PRD.md
Fase 3  RFC                  → docs/RFC.md (+ nomes de ADR congelados)
Fase 4  ADRs                 → docs/adrs/ (8 arquivos)
Fase 5  FDD (TDD)            → docs/FDD.md
Fase 6  Tracker              → docs/TRACKER.md
Fase 7  Reconciliação        → README.md + fechamento do PRD
Fase 8  Revisão final        → checklist de aceite item a item
```

### Duas escolhas de método que valem registrar

**1. Ordem de produção diferente da sugerida pelo enunciado.** O enunciado sugere *ADRs → RFC → FDD →
PRD*. Adotei **PRD → RFC → ADRs → FDD → Tracker → README**, que é a cadeia canônica de mercado — PRD
("what & why") → RFC ("how? let's debate") → ADR (decisão congelada) → tech spec — e coincide com a
tabela conceitual do próprio enunciado. A ordem sugerida é heurística de *redação* (as decisões já vêm
prontas da transcrição), não convenção de ciclo de vida.

Essa escolha cria uma dependência para trás: **o RFC precisa linkar ADRs que ainda não existem em disco**.
A mitigação foi congelar os nomes dos arquivos de ADR **antes** de escrever o RFC, e validar por `diff`
que os links do PRD e do RFC apontam exatamente para a tabela congelada.

**2. Uma fase inteira antes de escrever qualquer entregável.** A Fase 1 não produz nenhum documento da
entrega. Produz seis artefatos de triagem em [`docs/notas/`](./docs/notas/) — decisões fechadas,
requisitos, descartados, adiados, restrições numéricas e mapa do código — cada linha já com sua origem.
Todos os documentos posteriores são escritos **a partir desses artefatos**, não da transcrição bruta.
Isso é o que impede um item descartado de virar requisito três documentos adiante.

### Nota de nomenclatura: FDD = TDD

O enunciado chama de **FDD** ("Feature Design Document") o documento de implementação. No mercado, FDD
normalmente significa *Feature-Driven Development* (metodologia) ou *Functional Design Document* — e este
último descreve o sistema pela ótica de **negócio**, o oposto do que o enunciado pede. O que o desafio
chama de FDD é o que o mercado chama de **Technical Design Document / tech spec**. O nome do arquivo
segue `docs/FDD.md` por exigência da estrutura do entregável, mas o documento foi escrito com o modelo
mental de tech spec, e a equivalência está registrada no seu cabeçalho.

## Prompts customizados

A instrução do enunciado é explícita: prompts genéricos produzem documentos genéricos. Todo prompt de
extração pede **uma fatia específica**, com formato de saída definido e exclusões explícitas.

### Prompt 1 — extração por recorte, com exclusão explícita

Este é o prompt que separa o que entra do que não entra. Foram seis variações do mesmo molde, uma por
recorte (decisões fechadas / descartados / adiados / requisitos / números / menções ao código):

```
Da TRANSCRICAO.md, liste APENAS o que foi explicitamente DESCARTADO na reunião.

Para cada item:
- o que foi descartado
- o motivo declarado, com a frase que motivou o descarte
- o timestamp e o falante, no formato [hh:mm] Nome

NÃO inclua:
- decisões que foram fechadas
- itens adiados para uma fase futura (esses vão em outro recorte)
- ideias que apenas não foram mencionadas

Se um item foi levantado por uma pessoa e descartado por outra, registre os dois
timestamps. Não infira motivo que não tenha sido dito.
```

O ponto crítico é a seção **NÃO inclua**. Sem ela, "descartado" e "adiado" vêm misturados — e são coisas
diferentes: descartado vira *alternativa considerada* no RFC, adiado vira *questão em aberto*.

### Prompt 2 — verificação antes de citar

Aplicado ao mapa do código da Fase 1 e repetido como gate de cada documento seguinte:

```
Para cada arquivo, símbolo ou linha que este documento cita:

1. Confirme com grep/sed que o arquivo existe e que o símbolo está na linha alegada.
   Menção na reunião NÃO é prova de existência.
2. Separe em duas listas:
   - ÂNCORAS: existem hoje no repositório (cite com caminho:linha)
   - PROPOSTAS: foram mencionadas na reunião mas não existem (marque como "a criar")
3. Reporte qualquer citação que não resolva, com o valor correto.

Não conserte silenciosamente: liste o que estava errado.
```

O item 3 — *não conserte silenciosamente* — é o que transformou a verificação em ferramenta de
aprendizado: foi assim que os erros da seção seguinte apareceram.

### Prompt 3 — controle de altura entre documentos

Usado ao fim de cada documento, para impedir a duplicação que o enunciado penaliza:

```
Releia o documento que acabou de escrever contra a altura que lhe cabe:

- PRD: produto/negócio. ZERO endpoint, payload, schema de tabela ou código de erro.
- RFC: arquitetura. Propõe e abre para revisão; não desce ao detalhe do FDD.
- ADR: uma decisão fechada por arquivo, com o trade-off explícito.
- FDD: implementação. Referencia a decisão do ADR, não a reargumenta.

Liste os trechos que estão na altura errada e diga para qual documento cada um
deveria migrar. Se não houver nenhum, diga apenas isso.
```

## Iterações e ajustes

Foram **quatro correções concretas**, todas encontradas pela verificação mecânica, não por releitura.

### 1. Uma linha de código citada errada — e propagada por quatro fases

A Fase 1 registrou que a transação de `changeStatus` executa `tx.order.update` na **linha 156** de
`src/modules/orders/order.service.ts`. O FDD herdou a citação. Na verificação da Fase 5, o script que
imprime o conteúdo de cada linha citada mostrou:

```
src/modules/orders/order.service.ts  156  }
```

A chamada está na **linha 158**. Corrigido no FDD e na nota de origem.

**O que isso mudou no método:** a verificação da Fase 1 só validava referências no formato
`arquivo:linha` — números escritos em prosa, como "`tx.order.update` (156)", escapavam do `grep`. A
verificação passou a cobrir também as referências em prosa, e as outras 36 do pacote foram conferidas
uma a uma.

### 2. Caminhos que não existem, citados como se existissem

`src/modules/webhooks/` e `src/worker.ts` são mencionados na reunião com naturalidade, como se fossem
código. **Não existem.** A primeira versão do ADR-006 e do FDD os citava sem ressalva, o que colidiria
com o critério de aceite *"nenhum arquivo de código mencionado é inexistente no repositório"*.

Ajuste: os dois passaram a ser marcados explicitamente como **a criar**, e o FDD ganhou uma nota de
convenção logo no início — *todo caminho citado sem ressalva existe hoje e foi verificado; os caminhos a
criar são exatamente três*. O artefato `docs/notas/06-mapa-codigo.md` já mantinha essa separação numa
seção própria desde a Fase 1; o erro foi não propagá-la.

### 3. Números do próprio documento que não fechavam

A tabela-resumo do `docs/TRACKER.md` anunciava **183 itens rastreados**. A contagem programática das
linhas de dados deu **245**. O resumo foi refeito com os números apurados por documento e por fonte — e,
como o tracker é a defesa do pacote contra alucinação, um número errado *nele* seria especialmente ruim.

### 4. Um conflito entre dois gates, resolvido por registro em vez de corte

O enunciado pede que o RFC seja "conciso (2 a 4 páginas)". O RFC ficou em ~4,5–5 páginas — não por
verbosidade (a verificação confirmou zero payload, schema ou endpoint no documento), mas por **volume de
itens rastreáveis**: 7 alternativas descartadas, 5 questões em aberto e 5 riscos, todos com origem na
reunião.

Cortar para caber significaria remover itens que alimentam o tracker, cuja cobertura **é critério de
aceite pontuado** — enquanto a contagem de páginas **não aparece na checklist de aceite**. A escolha foi
manter a cobertura e registrar o desvio como decisão consciente, com a instrução exata de reversão —
reduzir §4 de 7 para 4 alternativas e §6 de 5 para 3 riscos — caso se prefira a faixa estrita.

### O que a verificação mecânica cobre hoje

Ao fim de cada fase, e de novo na revisão final:

- todo par `[hh:mm] Nome` existe literalmente em `TRANSCRICAO.md`, dentro de `[09:00]`–`[09:53]`;
- todo caminho de arquivo citado existe, e todo `arquivo:linha` aponta para o símbolo alegado;
- todo link markdown entre documentos resolve;
- os gates do tracker (cobertura, % de origem, contagens) são medidos, não estimados;
- nenhum valor numérico aparece fora do catálogo de `docs/notas/05-restricoes.md`;
- `git status` confirma que nada em `src/`, `prisma/`, `tests/`, configs, `TRANSCRICAO.md` ou
  `the-challenge/` foi tocado.

## Como navegar a entrega

### Ordem de leitura sugerida

| # | Arquivo | O que responde |
| --- | --- | --- |
| 1 | [`docs/PRD.md`](./docs/PRD.md) | Por que a feature existe, para quem, com que escopo e métricas |
| 2 | [`docs/RFC.md`](./docs/RFC.md) | Como propomos resolver, o que foi descartado e o que segue em aberto |
| 3 | [`docs/adrs/`](./docs/adrs/) | Por que cada decisão foi tomada exatamente assim — comece por [ADR-001](./docs/adrs/ADR-001-outbox-no-mysql.md) |
| 4 | [`docs/FDD.md`](./docs/FDD.md) | Como construir: fluxos, contratos, erros, integração com o código |
| 5 | [`docs/TRACKER.md`](./docs/TRACKER.md) | De onde veio cada coisa |

**Com pouco tempo?** [`docs/RFC.md`](./docs/RFC.md) §1 (TL;DR) dá a proposta inteira em um parágrafo, e
[`docs/FDD.md`](./docs/FDD.md) §10 mostra exatamente o que muda no código existente.

### Estrutura

```
.
├── README.md                    ← este documento (o processo)
├── TRANSCRICAO.md               ← insumo, não alterado
├── docs/
│   ├── PRD.md                   ← produto: por que e o quê
│   ├── RFC.md                   ← arquitetura: como propomos, o que está em aberto
│   ├── FDD.md                   ← implementação (tech spec): como construir
│   ├── TRACKER.md               ← rastreabilidade: 247 itens → transcrição ou código
│   ├── adrs/                    ← 8 decisões fechadas, formato MADR
│   │   ├── ADR-001-outbox-no-mysql.md
│   │   ├── ADR-002-retry-com-backoff-e-dlq.md
│   │   ├── ADR-003-hmac-sha256-com-secret-por-endpoint.md
│   │   ├── ADR-004-entrega-at-least-once-com-x-event-id.md
│   │   ├── ADR-005-worker-separado-em-polling.md
│   │   ├── ADR-006-reuso-dos-padroes-existentes.md
│   │   ├── ADR-007-snapshot-do-payload-na-insercao.md
│   │   └── ADR-008-filtragem-de-eventos-na-insercao.md
│   └── notas/                   ← material de processo (Fase 1), não é entregável
│       ├── 01-decisoes.md       ← 34 decisões fechadas, com timestamp
│       ├── 02-requisitos.md     ← 18 requisitos funcionais + 13 não funcionais
│       ├── 03-descartados.md    ← 18 itens descartados, com o motivo declarado
│       ├── 04-adiados.md        ← 10 itens adiados ou sem decisão
│       ├── 05-restricoes.md     ← 24 valores numéricos, marcando os ilustrativos
│       └── 06-mapa-codigo.md    ← 31 âncoras de código verificadas
├── src/  prisma/  tests/        ← aplicação existente, somente leitura
└── the-challenge/INDEX.md       ← enunciado original
```

> [`docs/notas/`](./docs/notas/) é **material de processo**, não entregável. Está versionado porque é a
> evidência de rastreabilidade por trás de cada documento: qualquer afirmação do pacote pode ser
> perseguida até a linha da transcrição ou do código que a originou.

### A aplicação existente

O código em `src/` é um Order Management System funcional (Node 20 + TypeScript + Express + Prisma/MySQL)
e serve apenas de **contexto e referência** — a entrega é puramente documental e **nenhum arquivo de
código foi alterado**. Os pontos do código que a feature toca estão mapeados em
[`docs/FDD.md`](./docs/FDD.md) §10, com 11 caminhos reais e o que muda em cada um.
