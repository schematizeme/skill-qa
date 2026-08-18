# Changelog — schematize-qa

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
