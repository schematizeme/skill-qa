# Execução — Plan-First, Test Kit, CI e Makefile

> Parte da skill **schematize-qa**. O **como se roda**: o fluxo plan-first (planeja → MD → aprovação →
> executa), o test kit (CLI único + saída machine-readable), o padrão de script, seeds/personas, a
> integração com CI (gates de teste/a11y/CWV) e o Makefile. Estratégia em `estrategia.md`; categorias
> em `categorias.md`; flaky em `flaky.md`.

## Índice
- 1. Fluxo de Q.A. plan-first (aprovação obrigatória)
- 2. Test kit e saída machine-readable
- 3. Padrão de script
- 4. Seeds e personas
- 5. Integração com CI (os gates)
- 6. Makefile padrão

---

## 1. Fluxo de Q.A. plan-first (aprovação obrigatória)

A malha de Q.A. inclui passos potencialmente destrutivos (chaos `service-kill`/`db-disconnect`,
mutação, carga). Por isso **nenhuma submissão de Q.A. roda às cegas**: toda submissão passa por
planejamento, aprovação humana e execução controlada. Comandos: `/qa-plan` (planeja) e `/qa-run`
(executa o aprovado).

**MUST — antes de executar qualquer coisa**
1. **Planejar tudo primeiro.** Ao receber a submissão, o agente **não executa nada ainda**. Levanta o
   escopo completo: quais camadas/modos vão rodar (unit/componente/integração/e2e/smoke/a11y/visual/
   contrato/dados/`simulated`/chaos), ambiente alvo, rotas e personas afetadas, ordem de execução,
   dependências entre passos, o que é **destrutivo/gated**, e os riscos.
2. **Gerar um MD de passo a passo detalhado** em
   `<projeto>_archive/qa/<YYYY-MM-DD-HH-MM-SS>-<contexto>.md` (archive obrigatório — §28 da
   engineering). Cada passo declara: objetivo, comando exato, ambiente, resultado esperado, critério de
   pass/fail, e flag de **destrutivo** quando aplicável. O plano referencia o `summary.json` (§2) que
   será produzido.
3. **Pedir aprovação explícita do usuário.** Sem aprovação registrada, **nada roda**. O agente
   apresenta o plano e aguarda o "ok". **Aprovação parcial** (subconjunto de passos) é válida e vira o
   escopo efetivo.

**MUST — após aprovado**
4. **Oferecer a modalidade de execução:**
   - **Faseado e assistido** — executa por fase, **pausa entre fases** para revisão/confirmação, mostra
     resultado parcial e só segue com o "continuar". Default recomendado para staging sensível e para
     qualquer plano com passo destrutivo.
   - **De uma vez (autônomo)** — executa o plano inteiro sem parar.
5. **No modo "de uma vez":**
   - **Multiagentes para a execução** — paralelizar categorias independentes (ex.: unit, a11y,
     `simulated`, contrato em workers separados), respeitando dependências declaradas e limites de
     concorrência (backpressure).
   - **Cron/watchdog de continuidade** — um agendador supervisiona e **retoma de checkpoint até
     concluir**, sem intervenção manual se um worker cair. "Conclusão" é condição de parada explícita:
     todos os passos aprovados terminaram, **ou** uma falha bloqueante escalou para o humano.
     Checkpoints são **idempotentes** e a retomada não reexecuta passo já concluído.

**MUST — segurança do fluxo**
- Passo destrutivo (`service-kill`, `db-disconnect`, drop, mutação em massa) **só roda se constava no
  plano aprovado** e com o gate de ambiente ligado (ex.: `<PROJECT>_CHAOS_ALLOW=1`). Aprovação do plano
  **não dispensa** o gate.
- Modo autônomo "de uma vez" roda por default só em `dev`/`staging`. Produção exige confirmação
  adicional explícita no momento da execução.
- **Sem retry infinito** no watchdog: limite de tentativas por passo, backoff, e escala pro humano ao
  estourar. "Ininterrupto até finalizar" é **retomar até concluir**, não repetir pra sempre (`flaky.md`).

**VETADO**
- Pular o plano ou a aprovação "pra ir mais rápido" — é macaquice (linha da §37 da engineering). Q.A.
  sem plano aprovado registrado **não roda**.

> O plano aprovado é o **contrato** da execução. Multiagente e cron aceleram o *como*, nunca dispensam o
> *o quê foi autorizado*.

---

## 2. Test kit e saída machine-readable

Toda a malha de testes "do sistema vivo" mora num repo dedicado (`<project>_ops`), invocada por um CLI
único. O **harness** (CLI, saída, seeds) é da casa; o conteúdo **ofensivo** dos modos de segurança é da
`schematize-pentest` (`categorias.md` §9).

**Estrutura padrão**
```
<project>_ops/
├── bin/<project>-test          # CLI: run modes, agrega saída
├── tests/
│   ├── lib.sh                  # helpers (scripts/lib.sh desta skill)
│   ├── README.md               # tabela de modos × scripts × duração
│   ├── smoke/  integration/  unit/            # Q.A. funcional (esta skill)
│   ├── a11y/  visual/  contract/  data/       # camadas transversais (esta skill)
│   ├── simulated/              # cobertura total de rotas × personas
│   ├── security/  pentest/  authz/  hardening/  chaos/   # segurança (schematize-pentest)
│   └── seeds/                  # SQL de setup (personas de teste, superadmin)
├── Makefile
└── README.md
```

**CLI padrão**
```bash
<project> test                  # default: smoke
<project> test unit | integration | e2e | smoke | a11y | visual | contract | data | simulated
<project> test all              # tudo exceto chaos+unit
<project> test full             # all + chaos + unit + mutation
<project> test seed-superadmin  # setup inicial (uma vez)
```

**Saída estruturada** — toda execução escreve em `/<project>/logs/test-<YYYY-MM-DD>-<HHMMSS-pid>/`:

| Arquivo | Conteúdo |
|---|---|
| `summary.txt` | PASS/FAIL por script, legível |
| `summary.json` | **fonte única pra dashboards/CI/gate** — schema fixo abaixo |
| `run-totals.txt` | mode, started, finished, duration, scripts, pass, fail, exit |
| `<mode>-<name>.log` | output completo de cada script |
| `coverage-summary.{txt,json}` | cobertura por serviço (só em `unit`) |

**Schema `summary.json`** (contrato — não quebre `mode`, `totals`, `scripts[]`):
```json
{
  "mode": "smoke",
  "started_at": "2026-05-11T11:35:50Z",
  "finished_at": "2026-05-11T11:35:57Z",
  "duration_seconds": 7,
  "log_dir": "/<project>/logs/test-2026-05-11-083550-448694",
  "scripts": [
    {"category": "smoke", "name": "auth-endpoints", "status": "pass"},
    {"category": "a11y",  "name": "checkout-axe",    "status": "fail"}
  ],
  "totals": {"pass": 23, "fail": 1, "scripts": 24, "exit_code": 1}
}
```

**MUST**
- Exit code 0 se tudo passou; 1 se qualquer caso falhou. `summary.json` é **o que o gate lê** (§5).
- Cada script é executável standalone (`bash tests/smoke/auth-endpoints.sh` sai `0/1`), declara
  `TEST_NAME`, usa `lib.sh`, e emite banner ENORME em vermelho quando falha.
- Skip **sem erro** só quando dependência **opcional** declarada falta. Logs zipados pra storage (≥ 90
  dias) após cada CI run.

---

## 3. Padrão de script (`tests/<mode>/<name>.sh`)

Skeleton obrigatório em `scripts/test-skeleton.sh` — status + shape do body + **assertion negativa**:

```bash
#!/usr/bin/env bash
# <Categoria> · <Nome curto> — o que cobre e o esperado.
set -uo pipefail
_DIR="$(cd -P "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
source "$_DIR/lib.sh"

test_section "<Categoria> · <Nome>"
API="$(api_base)"

assert_http_in "<caso>" "200|401" GET "$API/v1/<rota>"        # status

http_call GET "$API/v1/<rota>"                                # shape do body
if [[ "$HTTP_CODE" == "200" ]]; then
  echo "$HTTP_BODY" | jq -e '.data | length > 0' >/dev/null \
    && test_pass "<rota> retorna dados" \
    || test_fail "<rota> sem 'data'" "$HTTP_BODY"
fi

if echo "$HTTP_BODY" | grep -qiE 'stacktrace|exception|undefined|\{\{|\$\{'; then
  test_fail "<rota> vazou stack/placeholder" "$HTTP_BODY"     # assertion negativa
fi

test_summary "<categoria>/<nome>"
exit $TEST_EXIT_CODE
```

**MUST**: banner do `test_section` na 1ª linha; cada caso vira `✓`/`✗`/`○ skip`; `test_summary` agrega
contadores; falhas incluem `HTTP_BODY` truncado em 2000 chars (sem PII).

---

## 4. Seeds e personas

**MUST**
- Personas em `tests/seeds/test-*.sql` (ou `tests/<mode>/*.json`) — **versionadas, reproduzíveis**.
- Senhas test-only **explicitamente flagadas** (prefixo `SimTest!`) e **rejeitadas em produção** pelo
  validador de senha fraca.
- Setup **idempotente** (`INSERT ... ON CONFLICT DO UPDATE` — re-rodar não duplica). Cleanup só por
  comando explícito (`<project> test clean-seeds`), não automático (deixa lixo pra inspeção).

**Personas mínimas (RBAC + multi-tenancy):** `superadmin` (vê tudo), `tenant_admin_A`, `tenant_admin_B`
(pra testar isolamento), `normal_user` (só dados próprios). Opcionais: `viewer`/`editor`/`support`.

---

## 5. Integração com CI (os gates)

O gate dá dente à disciplina. Detalhe do que trava (e do que **não pode** desligar) em
`estrategia.md` §5.

**MUST — matriz de execução**
- **PR check** (rápido, ~3min): `smoke + unit afetado + a11y + lint`. Feedback em minutos.
- **Pré-deploy**: `<project> test all` (full minus chaos+unit, ~6min) + regressão visual + contrato.
- **Nightly**: `<project> test full` (inclui chaos + unit + **mutation** + carga).

**MUST — o que trava o merge**
- Fail no `summary.json` (`.totals.fail > 0`) **trava**.
- **a11y** (violação axe séria/crítica) **trava**.
- **CWV** fora do orçamento (LCP/INP/CLS acima do budget) **trava** (onde há frontend).
- Cobertura abaixo do mínimo (`estrategia.md` §3) **trava**. "No tests found" no caminho novo **trava**.
- Falha de smoke **pós-deploy** dispara **rollback automático**.

**VETADO**: `continue-on-error`, `allow_failure`, `--no-verify`, comentar o step, baixar threshold "só
nesse PR", ou mover o gate pra nightly "por enquanto" pra destravar um merge — o gate desligado não
volta (`estrategia.md` §5).

**Dashboards**: `summary.json` é parseado pra empurrar `tests_pass_total`, `tests_fail_total`,
`tests_duration_seconds_bucket`, e o **pass rate por teste** (detecção de flaky — `flaky.md` §2) pra
Prometheus.

---

## 6. Makefile padrão

```bash
make test             # unit + integration
make test-unit
make test-integration
make test-e2e
make test-a11y        # axe em componente + e2e
make test-visual      # regressão visual
make test-contract    # contratos consumer-driven
make smoketest        # smoke com self-check
make mutation         # mutation testing no domínio crítico
make coverage         # relatório de cobertura por camada
make ci               # tudo que o CI roda (gates inclusos)
```

> `make ci` é o que o gate roda — se passa local e passa no CI, é a **mesma** suíte. Divergência entre
> "verde na minha máquina" e "vermelho no CI" é ambiente não fixado (`flaky.md` §5), não azar.
