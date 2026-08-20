# Categorias de Teste (em detalhe)

> Parte da skill **schematize-qa**. Cada camada da pirâmide e cada camada transversal, com o que a
> torna uma guarda de verdade. Estratégia/proporção em `estrategia.md`; execução/test-kit em
> `execucao.md`. A matriz **adversarial** de ataque (injeção, IDOR, cross-tenant) é da
> **schematize-pentest** — aqui o `simulated` cobre **acessibilidade/cobertura de rota**.

## Índice
- 1. Unit — agressivo, não decorativo
- 2. Componente — comportamento, não markup
- 3. Integração — fluxos reais com credenciais/DB
- 4. e2e — jornada do usuário
- 5. Smoke — "verde de verdade"
- 6. Acessibilidade (a11y)
- 7. Regressão visual
- 8. Contrato e dados
- 9. Efeito externo — assertar e-mail/notificação SEM enviar de verdade
- 10. `simulated` — cobertura total de rota

---

## 1. Unit — agressivo, não decorativo

O grosso da malha. Rápido, isolado, sem rede/DB real. Delega ao runner nativo: Go `go test -race
-cover ./...`; TS `vitest run`/`jest`; Rust `cargo test`; etc.

**MUST**
- **`-race` sempre** (Go) / detector de concorrência ligado. Teste flaky por corrida = bug, não
  "re-roda" (`flaky.md`).
- **Caminho de erro obrigatório:** pra cada função, sucesso **e** falha (input inválido, dependência
  que erra, timeout, nil/null, vazio, divisão por zero, overflow).
- **Tabela de casos hostis por validador/parser:** tipo errado, fora do range, string gigante, vazia,
  unicode/RTL/null byte, número como string e vice-versa. Bug de sanitização morre no unit, antes de
  qualquer pentest achar.
- **Boundary obrigatório:** `0`, `-1`, `1`, `MAX`, `MAX+1`, vazio, um, muitos. O bug mora na borda.
- **Property-based** (SHOULD → MUST em domínio crítico): `fast-check` (TS), `gopter`/`rapid` (Go),
  `proptest` (Rust). Idempotência, round-trip (encode→decode), invariantes de agregado — acha o edge
  case que você não imaginou.
- **Mutation testing no domínio crítico** (`estrategia.md` §3): se o teste não pega a mutação, é
  decorativo.
- **Proibido teste que não pode falhar:** assert ausente, `expect(true)`, mock que devolve o próprio
  input esperado.

---

## 2. Componente — comportamento, não markup

Uma unidade de UI (ou um serviço + suas dependências próximas, mockadas na borda) exercitada como o
usuário/consumidor a usa. Ferramentas: Testing Library (React/Vue/Svelte), `@testing-library/user-event`.

**MUST**
- **Teste o COMPORTAMENTO observável, não a estrutura.** "Digitar e submeter mostra a mensagem de
  sucesso e limpa o campo" — não "o `<form>` existe" nem "o componente montou".
- **Consulte por papel/nome acessível** (`getByRole`, `getByLabelText`), **não** por classe CSS ou
  `test-id` frágil quando houver alternativa acessível. Isso faz o teste **também** exercitar a11y.
- **Interação real** (`user-event`, não só `fireEvent`): foco, tab, teclado, digitação — como um humano.
- **Sem snapshot cego** da árvore inteira. Snapshot, se usado, é de saída semântica pequena e revisada
  no diff.
- **Estados: loading, erro, vazio, sucesso.** Componente que só testa o estado feliz esconde o spinner
  eterno e o "erro" invisível.

---

## 3. Integração — fluxos reais com credenciais/DB

Fluxos ponta-a-ponta **dentro do serviço** com dependências reais (DB, cache, broker) — sem mock do
que se quer provar. Alvo: < 3min.

**MUST**
- **Credenciais reais + seed reproduzível:** login do superadmin com cookie real, CRUD ponta-a-ponta
  via API, upload, reset de senha completo. Usa `tests/seeds/*.sql` (idempotente — `execucao.md`).
- **O SQL/queries de verdade rodam** — é aqui que morre o bug que o mock esconde (coerção de tipo do
  driver, migration esquecida, índice ausente que só aparece com dado).
- **Idempotência e reprocessamento** onde o domínio exige (`Idempotency-Key` 2× → mesmo efeito).
- **Pré-seed declarado:** rodar `<project> test seed-superadmin` na primeira vez no ambiente.
- **Fluxo que notifica (cadastro, OTP, reset de senha, recibo) asserta a notificação LENDO DO SINK**
  (§9) — nunca de uma caixa real, nunca "presumindo que enviou".

---

## 4. e2e — jornada do usuário

Poucos, caros, alto valor. O usuário real atravessando o sistema montado (front + back + DB).
Ferramentas: Playwright (preferido), Cypress.

**MUST**
- **Jornadas, não telas:** "cadastra → confirma email → faz login → compra → recebe recibo", não
  "a página /checkout abre". O "confirma email" da jornada busca o link/código **no sink** (§9); e2e
  **nunca** abre uma caixa real nem espera um humano conferir.
- **Seletores por papel/label acessível** (`getByRole`, `getByLabel`) — resistem a refactor de CSS e
  exercitam a11y. `test-id` só quando não há nome acessível estável.
- **Determinismo** (`flaky.md`): espera por **condição** (elemento visível, request concluída), nunca
  `sleep(N)`; relógio/rede controlados; dados isolados por teste (sem depender de estado deixado por
  outro).
- **Poucos e valiosos:** e2e cobre os caminhos que **valem** a lentidão; o resto desce pra
  componente/unit.

---

## 5. Smoke — "verde de verdade"

Saúde do sistema pós-deploy, < 2min. **Falha = bloqueio de deploy.** Roda **antes e depois** de cada
`update` em todo ambiente. Um smoke que só confere `200` é teatro.

**MUST — anti "verde mentiroso"**
- **Assertar CONTEÚDO, não só status.** Toda rota-chave valida o **shape do body** (`jq -e`), não só o
  HTTP. `200` com body vazio, `{}`, `null`, `[]` onde deveria haver dado, ou HTML de erro com status
  200 = **FALHA**.
- **Assertion negativa obrigatória.** Cada rota crítica testa que o que **não** deveria estar lá não
  está: sem stack trace, sem `error`/`exception` no body de sucesso, sem `"undefined"`/`"null"`/`"NaN"`
  serializado, sem placeholder não renderizado (`{{`, `${`, `%s`).
- **Self-test do próprio smoke (meta-teste).** O suite tem um caso que **força uma falha conhecida**
  (bater numa rota fake `/_smoke_canary_should_404` esperando 404, e uma asserção que deve falhar de
  propósito num modo `--self-check`) pra provar que o runner **consegue reportar FAIL**. Se o
  self-check "passa" quando deveria falhar, o smoke está cego → CI quebra. Pronto em
  `scripts/smoke-selfcheck.sh`.
- **Cobertura de rota verificada.** O smoke compara as rotas que testou contra o inventário do
  OpenAPI/dispatcher. **Rota em produção sem caso de smoke = FALHA**, não silêncio.
- **Sem `|| true`, sem swallow.** Proibido `curl ... || true`, `set +e` sem voltar, ou condição que
  transforma erro em pass. Timeout numa dependência **obrigatória** é FAIL (skip só pra dependência
  **opcional** declarada).
- **Latência e dado fresco.** `/ready` valida dependência **de verdade** (ping no DB/broker), não
  `200` cacheado; o smoke afere `response-time p95` contra o SLO. Lento demais = FALHA.
- **Fail loud.** Banner vermelho ENORME (`lib.sh` → `test_summary`) e `summary.json` com
  `totals.fail > 0` travando o deploy.

> Smoke que nunca falha não é smoke saudável — é smoke quebrado.

---

## 6. Acessibilidade (a11y)

WCAG 2.2 AA é **piso**, não enfeite. a11y é **gate** (trava o merge), não relatório opcional.

**MUST**
- **axe automatizado** em componente e e2e (`@axe-core/playwright`, `jest-axe`, `axe-core`): zero
  violações **sérias/críticas**. Violação séria trava o merge como qualquer teste vermelho.
- **Teclado:** toda ação alcançável e operável só por teclado; ordem de foco lógica; **foco visível**;
  sem armadilha de foco. Testado por interação real, não presumido.
- **Nome acessível** em todo controle (botão, link, campo): `getByRole(...).name` existe e faz sentido.
  Ícone-botão sem `aria-label` = falha.
- **Contraste** AA (texto 4.5:1, UI 3:1) verificado; **não depender só de cor** pra transmitir estado.
- **Landmarks/headings/idioma** corretos; `prefers-reduced-motion` respeitado.

**SHOULD**
- Passe manual assistido (leitor de tela: NVDA/VoiceOver) nos fluxos críticos — o axe pega ~30-40% das
  barreiras; o resto é humano. Casa com o piso de a11y da `schematize-web`.

> a11y automatizada é chão, não teto. Verde no axe **não** significa acessível; significa "sem as
> barreiras que a máquina enxerga".

---

## 7. Regressão visual

Prova que a UI **não mudou de aparência sem intenção**. Ferramentas: Playwright screenshots,
Percy/Chromatic/Loki.

**MUST**
- **Baseline versionado e revisado.** O baseline é aprovado por humano; a atualização (`-u`) passa por
  **revisão do diff visual no PR**, nunca aceite automático em massa.
- **Determinismo** (`flaky.md`): fontes carregadas, animações desligadas (`prefers-reduced-motion` /
  freeze), dados/relógio fixos, viewport fixo. Screenshot flaky = quarentena, não threshold frouxo pra
  esconder.
- **Threshold consciente:** um antialiasing de 1px não é regressão; um layout deslocado é. O threshold
  é escolhido, não "o default que para de reclamar".
- **Escopo por componente** (preferido) além de páginas inteiras — diff pequeno é revisável; página
  inteira vira ruído.

---

## 8. Contrato e dados

**Testes de contrato (consumer-driven)**
- Onde serviços se integram, o **consumidor declara o contrato** que espera (Pact, ou schema/OpenAPI
  compartilhado) e o **produtor verifica** que o cumpre no seu CI. Muda o produtor, quebra o contrato,
  o pipeline do produtor falha — **antes** de quebrar o consumidor em produção.
- **Schema/OpenAPI como fonte:** validar request/response contra o schema; breaking change no shape
  exige **nova versão** (compat back/forward — casa com `schematize-data`).

**Testes de dados (expectations)**
- Todo dataset/pipeline valida **na borda**: tipos, ranges, obrigatoriedade, unicidade, integridade
  referencial, distribuição esperada (Great Expectations, `dbt test`, ou asserções próprias).
- **Dado ruim não contamina o downstream:** falha vira **quarentena/DLQ**, não silêncio nem crash.
  Idempotência e replay determinístico no reprocessamento. A disciplina de dados (contrato evolutivo,
  lineage, PII) é da **schematize-data**; aqui é o **teste** que a prova.

---

## 9. Efeito externo — assertar e-mail/notificação SEM enviar de verdade

Jornada real notifica: cadastro manda confirmação, login manda OTP, compra manda recibo, alerta manda
push/SMS/webhook. Testar isso é **obrigatório** — o bug do template quebrado (`{{nome}}` cru, link
apontando pra `localhost`) só aparece aqui. O que é **VETADO** é testar isso **enviando de verdade**:
o piso 10 da `SKILL.md` (normativa em `schematize-engineering` → `references/efeitos-externos.md`).

**A regra em uma linha:** o provider em teste é o **SINK**, e o teste **lê a caixa do sink por API**
(Mailpit `GET /api/v1/messages`) — nunca uma caixa real, nunca "olha lá no Gmail se chegou". Setup do
sink e variáveis em `execucao.md` §5.

### 9.1 O ciclo da asserção (integração e e2e)

1. **Limpar o sink** (`DELETE /api/v1/messages`) — isolamento por teste (`flaky.md` §3).
2. **Disparar a ação de domínio** pela API/UI (cadastrar, pedir reset, finalizar pedido).
3. **Esperar por CONDIÇÃO** — poll na API do sink até a mensagem do destinatário aparecer, com timeout
  curto e erro claro. **Nunca `sleep(N)`** (`flaky.md` §3): envio é assíncrono, `sleep` é flaky.
4. **Buscar por destinatário** (`?query=to:user+<run-id>-1@test.<domain>`), não "a última mensagem" —
  "a última" quebra na hora que a suíte roda em paralelo.
5. **Assertar conteúdo** (§9.2) e seguir a jornada com o que veio do e-mail (token, link, código).

**MUST**
- **Contagem exata:** a ação gera **1** mensagem, não "≥1". Duplicata (retry do worker mandando duas
  vezes) é bug que só esta asserção pega.
- **O OTP/token do e2e vem do sink**, não de uma caixa real e **não** de um "modo de teste que devolve
  o código na resposta da API" — expor OTP em response é buraco de segurança que vaza pra produção
  (`schematize-pentest`). O sink é o canal legítimo.
- **VETADO assertar "não deu erro no envio"** (`expect(sendMail).not.toThrow()`, `assert ok`): é teste
  tautológico (§1 e piso 1) — passa com o template quebrado, com o destinatário errado e com o corpo
  vazio.

### 9.2 Asserção de conteúdo do template

O e-mail é uma **saída do sistema** e se testa como qualquer outra: shape + conteúdo + assertion
negativa (mesma disciplina do smoke, §5).

| Assere | Exemplo |
|---|---|
| **destinatário exato** | `to == "user+r7f3a-1@test.<domain>"` — e o **domínio é o de teste** |
| **remetente e reply-to** | do subdomínio de envio do ambiente, não o de produção |
| **assunto** | bate com o esperado do template (e com o **idioma/locale** da persona) |
| **corpo — o dado que importa** | nome do usuário, valor do pedido, **código OTP no formato certo** |
| **link** | host = o do ambiente sob teste, token presente, e o link **funciona** (segue e valida) |
| **texto E HTML** | as duas partes existem e **contêm o mesmo dado** (multipart quebrado é clássico) |

**Assertion negativa obrigatória** — o e-mail **não** contém:
- **placeholder não renderizado**: `{{`, `${`, `%s`, `<%=`, `undefined`, `null`, `NaN`;
- stack trace, nome de exceção, caminho de arquivo do servidor;
- **segredo** (chave de API, hash de senha, token de sessão de terceiro) ou **PII além da necessária**;
- link pra `localhost`/host errado quando o ambiente não é local.

### 9.3 O teste NEGATIVO do guard (obrigatório — é o vermelho do guard)

O guard que recusa destinatário externo é uma **guarda**, e guarda vale pelo piso 1: **só conta se foi
vista reprovar**. Sem este teste, o guard é uma linha de código que ninguém sabe se funciona — e foi
exatamente assim que os 5.000 saíram.

```
teste "guard recusa destinatário externo fora de prd":
    dado  env = "hml", MAIL_PROVIDER = "sink", TEST_MAIL_DOMAIN = "test.<domain>"
    quando enviar(para: "alguem@gmail.com", template: "otp")
    então  ESPERA erro                      # assertion negativa: a chamada FALHA
      e    a mensagem cita o domínio de teste (mensagem acionável)
      e    o SINK está VAZIO                # nada saiu, nem pro sink
      e    o contador real_sent continua 0
```

**MUST — a matriz mínima do guard** (uma linha, um caso):

| Caso | `env` | Destinatário | Esperado |
|---|---|---|---|
| feliz | `test`/`hml` | `@test.<domain>` | **entrega no sink**, 1 mensagem |
| **guard (o vermelho)** | `test`/`hml` | `@gmail.com` | **ERRO**, sink **vazio** |
| fail-closed | *(config de env ausente)* | `@gmail.com` | **ERRO** (assume não-prd) |
| sem bypass | `test`/`hml` + flag `force=true` no chamador | `@gmail.com` | **ERRO** (guard não aceita bypass por parâmetro) |
| **cap** | `test` | N+1 envios com `MAIL_MAX_PER_RUN=N` | **abort** no N+1, com erro acionável |

- O caso do guard roda no **unit** (provider isolado, rápido) **e** é reexercido na **integração** com
  o provider real da composição — o guard tem que valer no objeto que o app de fato usa, não só no
  fake do teste.
- **Se o caso do guard passar a "passar" (envio aceito), o CI quebra** — como qualquer guarda que
  parou de guardar.

### 9.4 SMS, push e webhook

Mesmo padrão, mesmo oráculo: provider **sink** (número mágico do provedor / projeto de push de teste /
receptor HTTP local — **nunca** `webhook.site`, que é público, e nunca a URL do parceiro), asserção
lendo do sink, contagem exata, teste negativo do guard e cap por execução. SMS **custa por unidade**:
o cap importa mais, não menos.

> Um teste que "manda o e-mail de verdade pra ver se chega" não prova mais do que o sink prova — prova
> só que você tem uma caixa a menos e um domínio mais queimado.

---

## 10. `simulated` — cobertura total de rota

Engine (`scripts/simulated/run.py`) que cruza **rotas × personas × injections** e prova, por persona,
que **cada rota do inventário** responde como esperado — acessível pra quem deve, `403`/`401` pra quem
não deve.

**MUST — cobertura total (garantia de acessibilidade)**
- **Enumera 100% das rotas** do catalog (gerado do OpenAPI/dispatcher). **Rota no catalog sem resultado
  = FALHA** — não existe rota "não testada".
- **Reconciliação obrigatória:** rota servida em runtime mas ausente do catalog (fantasma) **e** rota
  no catalog que não responde (morta / `404` inesperado) **ambas** quebram o run. O nº de rotas
  testadas bate com o inventário.
- Toda rota é exercida com persona **autorizada e não autorizada** — acessibilidade e isolamento no
  mesmo passe.
- Saída: `raw.jsonl` (uma linha/request), `report.md` (seções **AUTO** e **REVIEW**), `summary.json`.
  Personas mínimas (`superadmin`, `tenant_admin`, `normal_user`) em `personas.json`.

> **Fronteira com a pentest:** o `simulated` daqui garante **cobertura e acessibilidade** de rota (Q.A.
> — "cada rota faz o certo pra quem deve"). A matriz **adversarial** — provar que cada campo **rejeita**
> injeção/coerção/cross-tenant, rota por rota — é da **schematize-pentest** (`/pentest-endpoints`,
> `/pentest-authz`). Reutilizam o mesmo catalog de rotas; mudam o oráculo.

**Modos de segurança no test kit** (`security`, `pentest`, `authz`, `hardening`, `chaos`): o **harness**
(CLI, saída, seeds) é compartilhado e mora no test kit (`execucao.md`), mas o **conteúdo ofensivo** de
cada um — os payloads, os critérios de rejeição, a severidade/gate — é **governado pela
schematize-pentest**. Q.A. roda o harness e coleta o resultado; a pentest define o que é ataque e o que
é PASS/FAIL de segurança.
