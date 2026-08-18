---
name: schematize-qa
metadata:
  version: 0.1.0
description: Q.A. / Quality Assurance da casa — a disciplina inteira de TESTE, agnóstica de linguagem, que prova que o software faz o que diz **com evidência**, não por fé. Rege a estratégia de teste e a pirâmide (unit → componente → integração → e2e), o princípio de que teste testa **COMPORTAMENTO** (o que o sistema faz), não "renderizou/montou" (teste tautológico é decoração); o "verde de verdade" — smoke com asserção de conteúdo + assertion negativa + **self-check que força uma falha conhecida** (smoke que nunca falha está cego); acessibilidade (axe/WCAG 2.2 AA) e **regressão visual** como testes que travam; testes de **contrato** (consumer-driven) e de **dados** (expectations/quarentena, casando com a schematize-data); **flaky tests** (detecção, determinismo e quarentena com prazo e dono — nunca silenciar pra sempre); **cobertura útil, não vaidade** (mínimos por camada + mutation testing; line coverage é piso, não meta; caminho de erro obrigatório); o fluxo **plan-first** (planeja o Q.A., gera um MD de passo a passo, **pede aprovação antes de rodar** — nada às cegas, passo destrutivo só com gate, watchdog que retoma de checkpoint sem retry infinito); os **gates de teste/a11y/CWV no CI que travam o merge** e NÃO se desligam "temporariamente"; e o Q.A. como parte inegociável da **Definition of Done**. Saída durável no `<projeto>_archive/qa/`. Use SEMPRE que for planejar, escrever, revisar, rodar ou auditar QUALQUER teste — unit, componente, integração, e2e, smoke, a11y, regressão visual, contrato, dados, property-based, mutation, carga/perf funcional — ou definir cobertura, montar o test kit, tratar teste flaky, configurar os gates de CI de teste, ou fechar uma entrega pela DoD — mesmo sem citar "teste" ou "Q.A.". Contém pisos inegociáveis (teste testa comportamento e é visto FALHAR no vermelho; cobertura é contrato — não se baixa a régua pra passar CI; smoke prova conteúdo, não status; a11y e flaky travam; Q.A. é plan-first e roda com aprovação; gate não se desliga "por enquanto"; archive obrigatório) que vetam o verde mentiroso. **Fronteiras:** segurança ofensiva (pentest de rejeição, injeção, IDOR/BOLA, cross-tenant, red-team) é a **schematize-pentest**; auditoria de histórico ("os checklists criados foram sanados?") é a **schematize-audit**; a engenharia base (DoD, archive, índice) é a **schematize-engineering** — esta skill é o **COMO** do teste que a engineering apenas exige.
---

# Q.A. / Quality Assurance da Casa (schematize-qa)

Disciplina normativa, **agnóstica de linguagem**, que responde a uma pergunta só: **o software faz o
que diz — e como você PROVA?** A casa não aceita "verde" por fé. Um teste só vale quando **testa
comportamento** (o que o sistema faz), foi visto **falhar no vermelho** quando o invariante quebra, e
o runner **o enxerga**. Tudo o mais — cobertura de linha, "renderizou", snapshot que ninguém lê — é
decoração até que o teste prove que reprova o caso ruim.

Esta skill foi **extraída da `schematize-engineering`** (as references `testes.md` + `testes-execucao.md`
e o comando `/eng-qa`) pra rodar solta e agnóstica de linguagem. A engineering mantém só o **piso
mínimo** — "a DoD exige testes verdes" — e delega o **COMO** pra cá. Onde a engenharia diz *que* tem
que ter teste, esta skill diz *como* o teste é escrito, rodado, medido e gated.

**Versão:** skill `schematize-qa` v0.1.0. Changelog em `CHANGELOG.md`.

## Comandos (Claude Code)

Digite `/qa-help` pra ver todos. Em resumo:

| Comando | O que faz |
|---|---|
| `/qa-help` | lista todos os comandos do schematize-qa |
| `/qa-plan` | **plan-first**: levanta o escopo (camadas/modos, ambiente, rotas/personas, destrutivos, riscos), gera o MD de passo a passo em `<projeto>_archive/qa/` e **pede aprovação ANTES de rodar** — nada às cegas |
| `/qa-run` | executa o Q.A. **do plano aprovado**: faseado/assistido (pausa entre fases) ou de uma vez (multiagentes + watchdog que retoma de checkpoint, sem retry infinito); passo destrutivo só com gate; coleta o `summary.json` e os gates de teste/a11y/CWV |
| `/qa-load` | carrega à força TODO o corpo normativo (estratégia/pirâmide, categorias, flaky, execução) e passa a aplicá-lo |
| `/qa-claude` | cria ou mescla o `CLAUDE.md` sempre-on de Q.A. na raiz do repo (não sobrescreve blocos de outras skills) |
| `/qa-cc` | context compact: gera handoff no archive e roda `/compact` |
| `/qa-handoff` | gera o handoff (context.md + checklist.md) **sem** compactar — pra fim de sessão |

Os comandos ficam em `assets/commands/` e são instalados em `.claude/commands/`.

## Como usar esta skill

1. **Plan-first sempre** (`/qa-plan`): a malha de Q.A. inclui passos potencialmente destrutivos
   (chaos, mutação, carga). **Nenhuma submissão de Q.A. roda às cegas** — planeja tudo, gera o MD,
   pede aprovação. Leia `references/execucao.md`.
2. **Escreva teste que testa COMPORTAMENTO** (`references/estrategia.md` + `references/categorias.md`):
   a pirâmide (unit → componente → integração → e2e) mais as camadas transversais (smoke, a11y,
   regressão visual, contrato, dados). Cada teste é visto **falhar no vermelho** (red-first) antes de
   valer como guarda.
3. **Meça o que o teste VERIFICA, não só o que executa** (`references/estrategia.md`): cobertura por
   camada é contrato (não se baixa a régua), line coverage é piso, mutation testing é a meta.
4. **Teste flaky é bug, não "re-roda"** (`references/flaky.md`): detecta, torna determinístico, ou
   coloca em quarentena **com prazo e dono** — nunca silencia pra sempre.
5. **Gate no CI trava o merge e NÃO se desliga "por enquanto"** (`references/execucao.md`): teste,
   a11y e CWV são bloqueantes; Q.A. é item da DoD.
6. **Não trabalhe de memória** — os limites por camada, o "verde de verdade" e o fluxo plan-first
   estão nos references. Aplique os pisos abaixo independentemente do reference carregado.

Mapa de references — leia o que casa com a tarefa:

| Tarefa | Reference |
|---|---|
| **Estratégia & pirâmide** (unit→componente→integração→e2e), teste testa **comportamento não "renderizou"**, guarda provada no vermelho (red-first), **cobertura útil não vaidade** (mínimos por camada + mutation), Q.A. como **DoD**, gates de CI que travam | `references/estrategia.md` |
| **Categorias em detalhe:** unit agressivo, componente, integração, e2e, **smoke "verde de verdade"** (conteúdo + assertion negativa + self-check), **a11y (axe/WCAG 2.2 AA)**, **regressão visual**, **contrato** (consumer-driven) e **dados** (expectations/quarentena), matriz `simulated` de acessibilidade de rota | `references/categorias.md` |
| **Flaky tests:** por que é bug (não "re-roda"), detecção (rodar N×, ordem aleatória, seed, `-race`), causas (tempo/sleep, estado compartilhado, rede real, relógio), **quarentena disciplinada** (prazo+dono, issue rastreável), determinismo | `references/flaky.md` |
| **Execução & gates:** o **fluxo plan-first** (planejar→MD→aprovação→rodar faseado/de-uma-vez, watchdog, destrutivo gated, sem retry infinito), **test kit** + CLI único + saída machine-readable (`summary.json`), padrão de script, seeds/personas, **integração com CI** (PR/pré-deploy/nightly, gate por `summary.json`, a11y/CWV), Makefile | `references/execucao.md` |

## Pisos inegociáveis (VETADO — sem ADR de exceção)

Independente do reference, estes limites nunca são cruzados:

1. **Teste testa COMPORTAMENTO — e é visto FALHAR no vermelho.** Um teste que verifica "renderizou/
   montou" ou que passa **sem nunca ter reprovado o caso ruim** não é guarda, é decoração. Escreva-o
   **red-first** (ou quebre o código de propósito uma vez) e confirme o vermelho. **Proibido teste que
   não pode falhar:** assert ausente, `expect(true)`, mock que devolve o próprio input esperado,
   snapshot cego. Revisão de PR rejeita teste tautológico.
2. **O runner tem que ENXERGAR o teste.** Confirme que os arquivos novos são **coletados** (globs de
   include cobrem `*.test.tsx`/`_test.go`/`tests/**`). Suíte verde que rodou **0** dos testes novos é
   falso-verde silencioso — falhe o CI em "no tests found" no caminho novo e/ou asserte a **contagem
   esperada** de casos.
3. **Cobertura é CONTRATO — não se baixa a régua pra passar CI.** Editar o threshold, pular teste
   (`.skip`), comentar assert ou baixar o mínimo de cobertura pra "passar o CI" é VETADO. Conserta o
   código, não o teste. Line coverage é **piso**, não meta; a meta é mutation score no domínio crítico.
4. **Smoke prova CONTEÚDO, não status.** `200` com body vazio/`{}`/`null`/erro-HTML = FALHA. Toda rota
   crítica valida o **shape do body**, tem **assertion negativa** (sem stack trace/placeholder), e o
   suite tem um **self-check que força uma falha conhecida** pra provar que o runner sabe reportar
   FAIL. Smoke que nunca falha está cego.
5. **Acessibilidade e regressão visual TRAVAM.** a11y (axe/WCAG 2.2 AA — teclado, foco, contraste,
   nome acessível) e regressão visual são gate, não relatório opcional. Violação de a11y bloqueia o
   merge como qualquer teste vermelho.
6. **Teste flaky é BUG, não "re-roda".** Flaky por corrida/tempo/estado é defeito. Detecta, torna
   determinístico, ou coloca em **quarentena com prazo e dono** (issue rastreável) — **nunca** silencia
   pra sempre nem esconde com retry infinito no gate.
7. **Q.A. é PLAN-FIRST e roda com aprovação.** Toda submissão de Q.A. planeja tudo, gera o MD de passo
   a passo e **pede aprovação antes de executar**. Passo destrutivo só roda se constava no plano
   aprovado **e** com o gate de ambiente ligado; autônomo só em dev/staging por default. Q.A. sem plano
   aprovado registrado **não roda**.
8. **Gate de CI trava o merge e NÃO se desliga "temporariamente".** Fail no `summary.json`
   (`.totals.fail > 0`), a11y vermelho, ou CWV fora do orçamento **travam o merge**. Desligar o gate
   "só por enquanto" é macaquice — o gate desligado não volta.
9. **Q.A. é parte da DoD e mora no archive.** Toda entrega passa pelos itens de teste da Definition of
   Done (§35 da engineering); todo plano/relatório de Q.A. mora em `<projeto>_archive/qa/` (§28), nunca
   no root. Sem archive, o Q.A. não aconteceu.

> Regra de bolso: se você **não viu o teste falhar de propósito**, você não sabe se ele funciona.
> Cobertura mede o que o teste **executa**; só o vermelho prova o que ele **verifica**.

## Andaime pronto (scripts e templates)

Não escreva do zero o que já está bundlado (movido da engineering pra cá):

- `scripts/lib.sh` — helpers de teste (`test_pass`, `test_fail`, `test_skip`, `test_section`,
  `test_summary`, `http_call`, `assert_http_in`, `api_base`). Todo script de teste usa estes.
- `scripts/test-skeleton.sh` — esqueleto obrigatório de `tests/<mode>/<name>.sh` (status + shape do
  body + assertion negativa).
- `scripts/smoke-selfcheck.sh` — o meta-teste anti "verde mentiroso": prova que o runner **consegue
  reportar FAIL** (normal=0, `--self-check`=1).
- `scripts/simulated/run.py` — scaffold do engine `rotas × personas × injections` (cobertura total de
  rotas: acessível pra quem deve, bloqueada pra quem não deve). O detalhe **ofensivo** das injeções é
  governado pela `schematize-pentest`; aqui o foco é **acessibilidade/cobertura de rota**.

## Relação com as outras skills

- **schematize-engineering** — a **BASE**. Ela **exige** teste (DoD §35, archive §28, índice §39) e
  delega o **COMO** pra cá. O piso "a DoD exige testes verdes" fica lá; a estratégia, as categorias, o
  flaky, a cobertura e o fluxo plan-first ficam aqui. As skills de linguagem (`schematize-go`/`rust`/
  `elixir`/`csharp`/`zig`/`ruby`/`web`/`node`) especializam o *runner* e a sintaxe; o piso de Q.A. é o
  mesmo em todas.
- **schematize-pentest** — a **fronteira de segurança ofensiva**. Provar rejeição rota-por-rota
  (injeção/coerção de tipo, IDOR/BOLA, BFLA, cross-tenant, SSRF, XSS, SQLi, JWT), threat model,
  hardening e red-team autorizado é **lá**, não aqui. O `simulated` desta skill cobre
  **acessibilidade/cobertura de rota**; a matriz **adversarial** de ataque é da pentest. Q.A. prova que
  o sistema **faz o certo**; a pentest prova que ele **rejeita o errado**.
- **schematize-audit** — a **fronteira de histórico**. Auditar se os **checklists criados foram
  sanados** ("feito" exige prova, órfãos RED) é lá. Q.A. gera evidência (teste que passa hoje); a audit
  **consome** essa evidência pra promover um `- [x]` de suspeito a fato.
- **schematize-data** — parceira nos **testes de dados**: expectations, qualidade na borda, quarentena/
  DLQ e contrato de schema evolutivo. O *como testar dado* referencia a disciplina de dados dela.
- **schematize-web** — parceira nos testes de **frontend**: componente (comportamento, não markup),
  e2e por seletor acessível, a11y e regressão visual casam com o piso de a11y/CWV do frontend.

## Aplicação sempre-on

Esta skill é puxada quando a tarefa casa com a descrição. Para garantir o piso em **toda** interação
do repo, copie `assets/CLAUDE.md` para a raiz do projeto (`/qa-claude` mescla sem sobrescrever os
blocos das outras skills). O `CLAUDE.md` pina o resumo e aponta pra cá; a skill entrega o detalhe e o
andaime.
