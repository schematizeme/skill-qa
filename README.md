# schematize-qa

> **Q.A. / Quality Assurance** da casa — a disciplina inteira de **teste**, agnóstica de linguagem,
> que prova que o software faz o que diz **com evidência**, não por fé. Um teste só vale quando
> testa **comportamento**, foi visto **falhar no vermelho**, e o runner **o enxerga**. O resto é
> decoração.

Pacote de **skill normativa para [Claude Code](https://claude.com/claude-code)**.
Parte do catálogo **schematize skills**. **Extraída da `schematize-engineering`** (as references
`testes.md` + `testes-execucao.md` e o comando `/eng-qa`) pra rodar solta e agnóstica de linguagem:
a engineering mantém só o **piso mínimo** ("a DoD exige testes verdes") e delega o **COMO** pra cá.

## Instalar

### Pelo app schematize (recomendado)

```bash
schematize install qa      # requer o CLI schematize instalado
```

### Última versão (a partir de um clone)

```bash
git clone https://github.com/schematizeme/skill-qa.git
cd skill-qa && ./install.sh                # instala no projeto atual
# ./install.sh /caminho/do/projeto          # ou aponte para outro projeto
# ./install.sh ~                            # global (~/.claude, todos os projetos)
```

Ou baixe o `.zip` da última release e descompacte em `.claude/skills/`:

```bash
curl -L -o skill-qa.zip \
  https://github.com/schematizeme/skill-qa/releases/latest/download/skill-qa.zip
unzip skill-qa.zip -d .claude/skills/
```

## O que tem dentro

- **SKILL.md** — o contrato: 9 pisos inegociáveis (teste testa comportamento e é visto falhar no
  vermelho; o runner enxerga o teste; cobertura é contrato — não se baixa a régua; smoke prova
  conteúdo, não status; a11y e regressão visual travam; flaky é bug com quarentena; Q.A. é
  plan-first com aprovação; gate de CI não se desliga "por enquanto"; Q.A. é DoD e mora no archive)
  + mapa de references + fronteiras com pentest/audit/data/web.
- **references/** — `estrategia` (pirâmide unit→componente→integração→e2e, comportamento não
  "renderizou", cobertura útil não vaidade + mutation, Q.A. como DoD, gates que travam),
  `categorias` (unit/componente/integração/e2e, smoke "verde de verdade", a11y WCAG 2.2 AA,
  regressão visual, contrato consumer-driven e dados/expectations, `simulated`), `flaky`
  (detecção + determinismo + quarentena com prazo e dono), `execucao` (fluxo plan-first, test kit,
  `summary.json`, seeds/personas, integração com CI, Makefile).
- **scripts/** — `lib.sh`, `test-skeleton.sh`, `smoke-selfcheck.sh`, `simulated/run.py` (o andaime
  de teste, movido da engineering).
- **assets/commands/** — `/qa-help`, `/qa-plan`, `/qa-run`, `/qa-load`, `/qa-claude`, `/qa-cc`,
  `/qa-handoff`.
- **assets/CLAUDE.md** — regra sempre-on do piso de Q.A.

## Regra de ouro

**"Verde" não é fé — é prova.** Um teste testa **comportamento** (o que o sistema faz), foi visto
**falhar no vermelho** quando o invariante quebra, e o runner **o enxerga**. Cobertura é contrato:
não se baixa a régua pra passar CI. Smoke prova conteúdo, não status 200. a11y e flaky travam.
Q.A. é **plan-first** (planeja → aprova → roda) e parte inegociável da **Definition of Done**.

## Relação com as outras skills

- **schematize-engineering** — a base que apenas **exige** teste na DoD (§35) e delega o **COMO** pra
  cá; archive (§28), índice (§39).
- **schematize-pentest** — a fronteira: segurança ofensiva (rejeição, injeção, IDOR/BOLA,
  cross-tenant, red-team) é dela, não desta skill.
- **schematize-audit** — auditoria de histórico ("os checklists criados foram sanados?") é dela.
- **schematize-data** — testes de dados/expectations/quarentena casam com a engenharia de dados.
- **schematize-web** — a11y e regressão visual do frontend compartilham o mesmo piso.

Co-autoria / patrocínio: Lucassa — https://lucassa.me

MIT.
