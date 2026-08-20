# Changelog — schematize-qa

## [0.2.0] — 2026-08-20

Propagação do piso **"efeito externo NUNCA sai de não-produção"** (normativa na
`schematize-engineering` → `references/efeitos-externos.md`) no recorte de Q.A. — a disciplina que mais
dispara efeito externo por acidente. Origem: incidente real de **>5.000 e-mails** disparados por um
laço de teste para endereços sintéticos, com reputação de IP e domínio queimada e utilidade zero.

### Adicionado
- **SKILL.md — piso inegociável 10: "Teste NUNCA dispara efeito externo real."** Endereço sintético só
  no **domínio de teste em ROTA NULA** (`<papel>+<run-id>-<n>@test.<domain>`; VETADO `@gmail.com`,
  caixa real, domínio de terceiro, e-mail de pessoa real — inclusive o seu — e o domínio de produção);
  provider = **SINK**; **guard DENTRO do provider** fail-closed; **cap por execução**
  (`MAIL_MAX_PER_RUN`); o **corolário de Q.A.** (conferir caixa em teste é **ler do sink por API**,
  nunca de caixa real); e a exigência de que o **guard tenha teste que vê o VERMELHO** (tenta
  `@gmail.com` com `env=hml` e **espera a recusa** — assertion negativa, red-first do piso 1). Registra
  que **carga/seed em massa multiplicam o efeito**: cap + sink separam "1.000 contas" de "1.000
  e-mails".
- **`references/execucao.md` §5 — "Efeito externo em teste — endereço, sink e cap"** (procedimento
  completo): endereçamento de seeds/personas/fixtures com tabela por papel e a lista de vetados; setup
  do **sink Mailpit** (SMTP 1025 + API HTTP 8025: `GET /api/v1/messages`, `GET /api/v1/message/{ID}`,
  `DELETE /api/v1/messages`) e o fake em memória pro unit; tabela de **variáveis do test kit**
  (`APP_ENV`, `MAIL_PROVIDER=sink`, `MAIL_SINK_API`, `MAIL_SINK_SMTP`, `TEST_MAIL_DOMAIN`,
  `MAIL_MAX_PER_RUN`, `TEST_RUN_ID`); **pré-flight fail-closed do runner** (6 verificações que abortam
  o run antes do 1º teste, incluindo o grep de caixa real em seeds/fixtures); regra de **carga/seed em
  massa** com cap coerente com o N; e o bloco **`external_effects` no `summary.json`** (`real_sent`,
  `blocked_by_guard`, provider, cap) que o gate confere.
- **`references/categorias.md` §9 — "Efeito externo: assertar e-mail/notificação SEM enviar de
  verdade"**: o ciclo da asserção (limpa o sink → dispara → espera por **condição**, nunca `sleep` →
  busca **por destinatário** → asserta), contagem exata (1, não "≥1"), OTP do e2e vindo do sink (e o
  veto a expor OTP na resposta da API), tabela de **asserção de conteúdo do template** (destinatário,
  remetente, assunto, dado, link, texto+HTML) com **assertion negativa** (`{{`/`${`/`undefined`, stack,
  segredo/PII, link pro host errado), o **teste NEGATIVO do guard** com a matriz mínima de 5 casos
  (feliz, guard, fail-closed, sem bypass por parâmetro, cap) e a extensão pra **SMS/push/webhook**.
- **`references/estrategia.md`** — novo item na **Definition of Done** (`real_sent == 0`, guard com
  teste vermelho, notificação assertada pelo sink) e novo item nos **gates de CI que travam o merge**.
- **`references/execucao.md` §6** — o gate de CI passa a travar em efeito externo real fora de prd
  (`external_effects.*.real_sent > 0`, provider real em job de teste, grep de caixa real em
  `tests/`/`seeds/`/`fixtures/`/`factories/`).
- **`references/flaky.md`** — envio real como causa de flakiness (fila/throttle/greylisting do
  provedor) na tabela de causas, e o sink como determinismo por design.
- **`assets/CLAUDE.md`** — novo piso sempre-on (7) com o resumo do veto + o ciclo de asserção pelo sink
  e o `real_sent == 0` no que o gate lê.

### Mudado
- **`references/execucao.md`** renumerado: "Integração com CI" §5 → **§6** e "Makefile padrão" §6 →
  **§7** (a nova seção de efeito externo entrou como §5, ancorada logo após seeds/personas).
- **`references/categorias.md`** renumerado: "`simulated` — cobertura total de rota" §9 → **§10** (a
  nova seção de efeito externo entrou como §9, junto das demais categorias).
- **`references/categorias.md`** §3 (integração) e §4 (e2e) passam a exigir explicitamente que fluxo
  que notifica asserte a notificação **lendo do sink**; §4 (seeds/personas) da `execucao.md` ancora o
  endereço das personas no §5.

Formato: [Keep a Changelog](https://keepachangelog.com/pt-BR/). Versionamento semântico.

## [0.1.0] — 2026-08-18

Primeira versão da skill de **Q.A. / Quality Assurance** da casa — a disciplina inteira de teste,
extraída da `schematize-engineering` (references `testes.md` + `testes-execucao.md` e o comando
`/eng-qa`) pra rodar solta e agnóstica de linguagem. A engineering passa a manter só o piso mínimo
("a DoD exige testes verdes") e delega o COMO pra cá.

### Adicionado
- **SKILL.md** com 9 pisos inegociáveis (teste testa comportamento e é visto falhar no vermelho; o
  runner enxerga o teste; cobertura é contrato — não se baixa a régua; smoke prova conteúdo; a11y e
  regressão visual travam; flaky é bug com quarentena; Q.A. é plan-first com aprovação; gate de CI não
  se desliga "por enquanto"; Q.A. é DoD e mora no archive) + mapa de references + fronteiras com
  pentest/audit/data/web.
- **references/**:
  - `estrategia.md` — a pirâmide (unit→componente→integração→e2e) + camadas transversais; teste testa
    **comportamento não "renderizou"** (anti-tautológico, guarda provada no vermelho, runner enxerga);
    **cobertura útil não vaidade** (mínimos por camada + mutation testing); Q.A. como **DoD**; gates de
    CI que travam e não se desligam.
  - `categorias.md` — unit agressivo, componente (comportamento não markup), integração, e2e, **smoke
    "verde de verdade"** (conteúdo + assertion negativa + self-check), **a11y (axe/WCAG 2.2 AA)**,
    **regressão visual**, **contrato** (consumer-driven) e **dados** (expectations/quarentena),
    `simulated` (cobertura total de rota) — com a fronteira explícita: matriz adversarial é da pentest.
  - `flaky.md` — flaky é bug (não "re-roda"); detecção (rodar N×, ordem aleatória, `-race`, CI que
    marca); causas + conserto; **quarentena disciplinada** (prazo+dono, issue, teto, saída provada);
    determinismo por design.
  - `execucao.md` — o **fluxo plan-first** (planejar→MD→aprovação→rodar faseado/de-uma-vez, watchdog,
    destrutivo gated, sem retry infinito); **test kit** + CLI único + saída machine-readable
    (`summary.json`); padrão de script; seeds/personas; **integração com CI** (matriz PR/pré-deploy/
    nightly, gates de teste/a11y/CWV que travam); Makefile.
- **assets/commands/**: `/qa-help`, `/qa-plan` (plan-first, o que era `/eng-qa`), `/qa-run` (executa o
  aprovado), `/qa-load`, `/qa-claude`, `/qa-cc`, `/qa-handoff`.
- **assets/CLAUDE.md** — regra sempre-on: teste testa comportamento e é visto falhar; cobertura é
  contrato; smoke prova conteúdo; a11y/flaky travam; Q.A. é plan-first e DoD.
- **scripts/** (movidos da engineering): `lib.sh`, `test-skeleton.sh`, `smoke-selfcheck.sh`,
  `simulated/run.py`.

### Contexto da extração
- A `schematize-engineering` teve as references `testes.md`/`testes-execucao.md` e o comando `/eng-qa`
  removidos; no lugar ficou um ponteiro curto pra cá + o piso mínimo na DoD. As referências cruzadas
  de teste (`§22.x`) na engineering foram repontadas: teste/cobertura/smoke → `schematize-qa`;
  hardening/pentest/injeção → `schematize-pentest`.
