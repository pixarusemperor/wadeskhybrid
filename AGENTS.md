# AGENTS.md — DeskcommCRM

> Contrato para **qualquer** agente de código (Codex, Cursor, Copilot, Amp, Claude Code).
> Este arquivo é o núcleo portável. A **doutrina completa e não-negociável vive em
> [`CLAUDE.md`](CLAUDE.md)** — leia-o antes de tocar em código. Aqui está o mínimo
> para não causar dano.

---

## Objetivo do projeto

Sistema operacional de vendas open source com agentes de IA nativos, multi-nicho,
WhatsApp como canal primário (via WAHA). Multi-tenant com RLS desde o dia 1, LGPD
nativa. Monetização = self-host em VPS, não assinatura. Posicionamento: [`VISION.md`](VISION.md).

**Consequência que muda como você trabalha:** o produto é distribuído como código.
Quem instala numa VPS **é** o usuário. Uma mudança que funciona na máquina do dev e
quebra no clone fresco é um bug de produto, não um detalhe de ambiente.

## Stack (CONFIRMADO em `package.json`)

Next.js 16 (App Router) · React 19 · TypeScript 6 estrito · Tailwind 3 ·
shadcn/ui · Supabase (Postgres + Auth + Realtime + Storage) · Upstash Redis ·
Vercel AI Gateway (`@ai-sdk/anthropic|openai|google`) · WAHA Plus (engine NOWEB) ·
Zod 4 · Vitest 4 · Playwright 1 · Sentry 10.

Só a **major**, de propósito: é onde o idioma muda, e é o que
`tests/unit/agents-md-versoes.test.ts` verifica contra o `package.json`. Declarar a minor
aqui fazia todo bump do Dependabot reprovar o `verify` (5 dos 8 pacotes) e não cobria nada
que a major já não cobrisse — issue #235. Para a versão exata, `package.json` é a fonte.

Runtime: **Node ≥22** (`.nvmrc` = 22; os quatro workflows fixam `node-version: 22` —
`ci` ×2, `perf`, `e2e`). Gerenciador: **pnpm 9.15.9** (`packageManager`).
Versão do produto: **1.0.0** (`CHANGELOG.md`, SemVer — mudança que afeta quem roda VPS entra lá).

## Estrutura que importa

| Path | O quê |
|---|---|
| `app/api/v1/` | 166 route handlers REST (versionado por path) — 169 contando `app/api/**` |
| `app/api/internal/`, `app/api/mcp/`, `app/api/v1/cron/` | superfícies não-cookie (secret/bearer próprio) |
| `app/app/` | UI autenticada do tenant · `app/admin/` UI de plataforma |
| `app/actions/` | Server Actions (auth, onboarding, team, settings) |
| `lib/agent-engine/`, `lib/ai/` | runtime do agente, guardrails, RAG, dispatcher |
| `lib/api/wrappers.ts` | `ok()` / `fail()` — **use sempre**, não monte Response na mão |
| `lib/auth/require-role.ts` | `requireRole()` — guard canônico de RBAC |
| `lib/supabase/{browser,server,admin}.ts` | clients canônicos |
| `workers/` | workers de `event_log` + crons |
| `supabase/migrations/` | schema versionado · `supabase/baseline.sql` = o que o self-host aplica |
| `proxy.ts` | middleware do Next 16 (auth de borda, `X-Request-Id`) |

## Comandos (CONFIRMADO em `package.json`)

```bash
pnpm install          # deps (frozen-lockfile no CI)
pnpm dev              # dev server
pnpm build            # next build
pnpm lint             # eslint
pnpm typecheck        # tsc --noEmit (estrito)
pnpm test:unit        # vitest — EXCLUI tests/invariants e tests/e2e
pnpm test:db          # invariantes de banco + gate do baseline (PRECISA de Docker)
pnpm test:e2e         # Playwright (PRECISA de app rodando + banco semeado)
pnpm gov:verify       # typecheck + lint + test:unit  ← verificação única atual
```

⚠️ **`pnpm gov:verify` NÃO cobre tudo.** Ele omite `test:db` e `test:e2e`. Se sua
mudança toca schema, RLS ou UI, `gov:verify` verde **não** é prova — rode `pnpm test:db`
(exige Docker) e/ou `pnpm test:e2e` você mesmo. Ver [`docs/harness-audit.md`](docs/harness-audit.md).

**O que o CI cobre.** `.github/workflows/ci.yml`: `verify` = typecheck + lint + test:unit;
`invariants` = `pnpm test:db` (isolamento RLS + invariantes de governança contra Postgres
efêmero pg17). `.github/workflows/perf.yml`: `build-and-size` = `pnpm build`.
`.github/workflows/e2e.yml` roda **45 das 46 specs** Playwright contra um Supabase local de
verdade com o `baseline.sql` aplicado — o mesmo banco que o self-hoster tem. **É check
obrigatório desde 2026-08-08.** A **única** de fora é `vps-fresh-onboarding` (WAHA + Redis +
Resend + Nuvemshop; é a P0 da doutrina de QA) — ou seja, `e2e` verde não prova a jornada de
instalação fresca. `followup-journey`, `webhooks` e `capacidades-do-agente` estiveram fora e
**voltaram**: rodam hoje (`e2e.yml`, listas `SPECS_PARTE_1`/`SPECS_PARTE_2`).

`.github/workflows/publish-image.yml`: `imagens-ok` = as três imagens Docker constroem. **Obrigatório
desde 2026-08-13.**

**Os cinco são checks obrigatórios** na branch protection da `main` — medido em 2026-08-14 @ `741c4ec8`:

```console
$ gh api repos/melgarafael/DeskcommCRM/branches/main/protection --jq '.required_status_checks.contexts|join(", ")'
verify, build-and-size, invariants, e2e, imagens-ok
```

> Este bloco estava errado em quatro pontos até 2026-08-14 (dizia "três checks", "28 das 32
> specs", "e2e não é obrigatório ainda" e listava como excluídas três specs que já rodavam).
> A pior era a do `e2e`: quem lesse mediria um PR contra a régua errada. **Reconte antes de
> citar** — `ls tests/e2e/*.spec.ts | wc -l` e o comando acima.

## Padrões de código (observados no repo, não inventados)

- **Route handler:** valida input com Zod → guard (`requireRole` / `requirePlatformAdmin` /
  secret) → query com `organization_id` explícito → `audit()` se mutação → `ok()` / `fail()`.
- Erro: `fail(code, message, status)` com código de `lib/api/errors.ts`. Nunca `throw` cru na borda.
- JSON **snake_case** na API. Dinheiro em `_cents` + `currency`. Datas ISO-8601 UTC.
- Log: `lib/logger.ts` (estruturado). **`console.log` é proibido** em código merged.
- Testes ao lado do código (`lib/foo/bar.test.ts`) ou em `tests/{unit,api,invariants,e2e}/`.
- Comentários em PT-BR são a norma neste repo — mantenha o idioma do arquivo que editar.

### Marca própria (white-label) — o produto é revendido, e o nome não é seu

- **Nunca escreva "Deskcomm"/"DeskcommCRM" em código que alcança o usuário.** `tests/unit/branding.test.ts` varre `app|components|lib|workers|hooks` e reprova; a allowlist **só encolhe**.
- A marca resolve do **banco** (`platform_branding` para a instalação, `organizations.settings.branding` para a organização). `APP_NAME`/`APP_LOGO_URL`/`APP_ACCENT_HEX` no `.env` são **semente e piso de rollback**, não a fonte.
- Precisa da marca **fora do DOM** (e-mail, remetente, ícone, `issuer` do MFA)? Use `marcaDaSaida()` de `lib/branding/saida.ts` — um hex e uma frente legível, tema claro. Nunca entregue `MarcaResolvida` a um template de e-mail.
- Resolvedor de marca **nunca lança**: ele roda em `app/layout.tsx`, e um throw ali é 500 em todas as telas.
- **O PDF de LGPD não leva marca** — ele nomeia o controlador (`organizations.legal_name`) e o DPO. Isso é decisão, não omissão; há gate no mapa de arquitetura.
- Contexto de venda em `docs/white-label.md`; mapa em `docs/architecture/marca-propria.architecture.json`.

## Diretórios e arquivos SENSÍVEIS

- **`supabase/baseline.sql`** — é o que o `install.sh`/`update.sh` do self-host aplicam.
  Toda mudança de schema tem que aparecer aqui **como apêndice idempotente**, senão
  não chega em quem instalou. Ver doutrina de Migrations em `CLAUDE.md`.
- **`supabase/migrations/*.sql` já aplicadas** — nunca edite. Corrija com migration nova.
- **`lib/supabase/admin.ts`** — service role **bypassa RLS**. 89 rotas o usam; toda
  query precisa filtrar `organization_id` manualmente, resolvido de fonte confiável
  (cookie/JWT/webhook secret/path token), **nunca do body**.
- **`lib/auth/public-paths.ts`** — adicionar path aqui remove a checagem de auth de borda.
  Só com guard próprio dentro da rota.
- **`.env*`** — não abra, não copie valor, não logue. Só `.env.example` é template.
- **`docker-compose.traefik.yml`** — numa VPS que já tem proxy reverso próprio
  (Hostinger, Coolify, Dokploy…), é o único lugar que dá ao contêiner `app` as labels
  de roteamento. Todo `up -d` leva os **dois** arquivos de compose:
  `docker compose -f docker-compose.prod.yml -f docker-compose.traefik.yml --env-file .env up -d app`.
  Esquecer o segundo `-f` recria o contêiner sem labels: o proxy deixa de enxergá-lo e o
  domínio inteiro responde `404`, com o contêiner `healthy` — o healthcheck é um probe TCP
  interno e não sabe nada de roteamento. Runbook: `docs/runbooks/deploy.md`.

## Arquivos GERADOS — não editar à mão

- `lib/database.types.ts` (6.1k linhas — gerado do schema Supabase)
- `graphify-out/` (grafo de conhecimento; regenerado por `/graphify .`)
- `pnpm-lock.yaml`, `tsconfig.tsbuildinfo`, `next-env.d.ts`, `.next/`

## Como validar uma alteração

1. `pnpm typecheck` e `pnpm lint` zerados.
2. `pnpm test:unit` verde.
3. Tocou schema/RLS/tabela tenant-aware → `pnpm test:db` (sobe Postgres efêmero via Docker,
   aplica `baseline.sql` em modo install **e** update, roda os invariantes).
4. Tocou UI ou fluxo de usuário → `pnpm test:e2e` com evidência visual. **`curl` não conta**
   como prova de UX (doutrina de QA Visual em `CLAUDE.md`).
5. Mudou schema → migration versionada em `supabase/migrations/` **+** apêndice idempotente
   em `supabase/baseline.sql` **+** linha em `supabase/migrations/MANIFEST.md`. Os três juntos.
6. Criou função em `public` → `revoke execute on function ... from public, anon;` e depois
   `grant` só a quem precisa. São **duas** origens de `EXECUTE` e revogar uma só deixa a
   função exposta como RPC alcançável pela anon key. Detalhe em `CLAUDE.md`, item 9 da
   doutrina de Migrations.

## Testes existentes (CONFIRMADO)

Medido em 2026-08-14 @ `741c4ec8`, com o comando ao lado de cada número:

- **257** arquivos de teste unitário em `tests/unit/` (`git ls-files 'tests/unit/*.test.ts' 'tests/unit/*.test.tsx' | wc -l`). O repo tem **491** arquivos `*.test.ts(x)` no total (`git ls-files '*.test.ts' '*.test.tsx' | wc -l`) — a diferença vive junto ao código, fora de `tests/`, e também roda em `test:unit`.
- **102** arquivos de invariante de banco em `tests/invariants/` (`git ls-files 'tests/invariants/*.test.ts' | wc -l`) — RLS/isolamento cross-tenant,
  RBAC, governança (G1–G6). Excluídos do `test:unit` de propósito; rodam via `pnpm test:db`
  **e no job `invariants` do CI**.
- **46** specs Playwright em `tests/e2e/` (`ls tests/e2e/*.spec.ts | wc -l`). **45 rodam no CI** (via `e2e.yml`,
  **obrigatório**). A única de fora é `vps-fresh-onboarding`, por dependência de serviço externo
  (WAHA/Redis/Resend/Nuvemshop). Ver issue #63.

## Limitações conhecidas (estado em 2026-07-29, contra `origin/main` @ 789dfa6)

- **1 das 46 specs E2E segue fora do CI** (`vps-fresh-onboarding`), e o `e2e` **é** check
  obrigatório desde 2026-08-08. Ou seja: um PR que quebre o `e2e` não entra — mas a jornada de
  instalação fresca, que é o produto que se vende, continua sem gate. Se você mexeu nela, a
  prova é sua. *(Corrigido em 2026-08-14; a redação anterior — "4 das 32, não-obrigatório" —
  mudava a régua de qualquer triagem que a lesse.)*
- Rate limit HTTP: `lib/auth/rate-limit.ts` cobre **login, signup, recuperação de senha e
  aceite de convite** (contando por IP **e** por identificador hasheado); `checkRateLimit` cobre
  o webhook de captação e o dispatcher de IA. **Crons e MCP seguem sem.** Meça antes de agir:
  `grep -rln 'authRateLimited\|checkRateLimit(' app lib --include='*.ts' --include='*.tsx'`.
  Esta linha dizia "existe em 2 pontos; login e signup estão sem" — era o estado anterior à
  issue #64, e o `docs/threat-model.md` ainda carrega a versão velha, com nota de reauditoria.
- Fallback do rate limit é **em memória** — sem Upstash configurado o limite é por processo.
- `Idempotency-Key` implementado em **1** rota, apesar de o contrato prometer nos POSTs de criação.
- **`.env.example` está completo** — medido em 2026-08-14: das 45 chaves de `lib/env.ts`, a
  única ausente é `NODE_ENV`, que não é configuração do operador. Esta linha dizia que faltavam
  6, "incluindo 3 secrets"; os três (`IMPERSONATE_COOKIE_SECRET`, `INTERNAL_CRON_SECRET`,
  `LGPD_SIGNING_KEY`) estão lá. Se você adicionar env var, adicione nos dois lugares (item 9 do
  DoD) — a regra continua valendo, o que caiu foi a dívida.
- `lib/auth/invite-token.ts` cai em `"dev-fallback"` como secret HMAC se nenhum secret existir
  (inalcançável em produção, porque `INTERNAL_SECRET` é obrigatório e derruba o boot).
- **89 dos 169 handlers de `app/api/**` usam service role** — sem gate automático para o filtro de
  `organization_id`. Escrevendo handler novo, o filtro é responsabilidade sua.
- Detalhes e prioridade: [`docs/harness-audit.md`](docs/harness-audit.md),
  [`docs/current-state.md`](docs/current-state.md) e [`docs/threat-model.md`](docs/threat-model.md).

## Regras de segurança

- Sempre `getUser()` no backend. **Nunca `getSession()`** (confia no cookie sem revalidar).
- API key/token **nunca** em query string — só header. Plaintext do bearer é mostrado
  **uma vez**; no banco só hash SHA256.
- HMAC de webhook com `crypto.timingSafeEqual`. Fail-closed quando o secret falta.
- Nunca logue segredo, token, CPF, telefone ou e-mail. Sentry tem `beforeSend` que
  higieniza — não confie nele como única camada.
- Não commite screenshot/dump com dado real de cliente.

## Packaging — se você tocou `Dockerfile*`, `docker-compose*.yml` ou `hostgator-setup-kit/`

Lei completa em [`docs/doctrine/packaging.md`](docs/doctrine/packaging.md). O não-negociável:

- **Nenhum serviço de `docker-compose.prod.yml` constrói na máquina do cliente.** Todo serviço
  declara `image:` de uma imagem publicada; `build:` só existe **ao lado**, como escape.
  Serviço `build:`-only é pulado por `docker compose pull` e imune a `up -d` sem `--build` —
  ele não é só caro de instalar, ele **nunca é atualizado**.
- **Publicação é ato do CI**, nunca da sua máquina: build ARM local não roda na VPS amd64.
- **Instalação de cliente aponta para número de versão**, nunca para tag móvel. Aqui `latest`
  significa **topo da `main`**, não última release — quem quer a última release usa `stable`.
- **Dependência upstream é referenciada com tag fixa, nunca republicada** (WAHA é licenciado).
- **Bump de versão não pode exigir que o operador da VPS edite arquivo à mão.**

`pnpm test:shell` é o único gate que exercita o kit. Rode-o.

## Critério de conclusão

Vale a **Definition of Done em [`CLAUDE.md`](CLAUDE.md)** — conte lá em vez de confiar num número aqui (`sed -n '/^## Definition of Done/,/^Um staff engineer/p' CLAUDE.md | grep -cE '^[0-9]+\. '`; esta linha já disse 15 e o DoD tem 16). A régua tem que DELIMITAR a seção: a primeira versão desta linha oferecia `grep -c '^[0-9]\+\. \*\*' CLAUDE.md`, que devolve **25** — casa toda linha numerada em negrito do arquivo (anti-patterns, packaging, higiene de branches, migrations) e perde os itens 1–10 do próprio DoD, que não são negrito. Trocar o número pelo comando só ajuda se o comando responder à pergunta. Não declare pronto
sem: typecheck/lint zerados, testes relevantes verdes, RLS testada se tocou tabela
tenant-aware, migration + baseline + MANIFEST se mudou schema, prova visual se mudou UI, e a
regra de packaging acima se mudou o artefato que o self-hoster instala.

## Regra final — não invente

Este repositório tem PRDs, specs, regras de negócio e doutrina escritos
(`docs/prd/`, `docs/specs/`, `docs/business-rules/`, `docs/doctrine/`).
**Nunca invente regra de negócio, número, SLA ou comportamento de produto.**
Se a regra não está escrita, diga que não está e pergunte — não preencha a lacuna com
suposição plausível. Ao documentar, marque o que é `CONFIRMADO` (provado por código) e o
que é `INFERIDO`.

<!-- BEGIN:nextjs-agent-rules -->

# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` (resolved from this file's directory; in monorepos the `next` package may not be visible from the repo root) before writing any code. Heed deprecation notices.

This block is written and re-added by `next dev` — verify at `node_modules/next/dist/server/lib/generate-agent-files.js`. Removing it from a diff only re-creates the uncommitted change; committing it with your work keeps the tree clean.

<!-- END:nextjs-agent-rules -->

## Restrições do PROJECT DIRECTION.md

[`PROJECT DIRECTION.md`](PROJECT%20DIRECTION.md) define restrições **não-negociáveis** para esta implementação. Antes de tocar em código, consulte-o integralmente.

### Proibido

- **Rebuild de foundations desnecessário** — preserve o que já funciona. Cada modificação deve responder: por que não pode ser implementado ao lado?
- **Requerer AI API key para V1** — AI é opcional. Não remova código AI existente. Não faça workflow execution depender de AI. Não hardcode outro provider.
- **Confundir PAT e Session API Key** — PAT é account-level (server-side only); Session Key é session-scoped. Nunca exponha nenhuma no browser (`NEXT_PUBLIC_*`), nunca logue, nunca commite.
- **Duplicar sistema de idempotency** — reutilize a infraestrutura existente. Evento duplicado = 1 processamento, 0 mensagens duplicadas de saída.
- **Executar workflow longo no webhook** — verify signature → ack → persist → queue. Never synchronous.
- **Introduzir novo database** — use Supabase. Cloudflare R2 proibido em V1 a menos que audit proving Supabase Storage não suporta.
- **Copiar outro CRM** — apenas componentes UI do WA CRM builder (MIT), adaptados. Nunca Meta/WhatsApp-specific code do WA CRM.
- **Pular phase gates** — STOP on failure. Nunca proceed enquanto a fase atual falha. Checkpoint = implemented + tested + verified.
- **Fazer testes dependerem de delays reais** — use fake clocks / time injection.
- **Bypassar o CRM para produção** — agentes externos não chamam Wasender `send-message` diretamente para campaign execution. Todo envio passa pelo CRM.
- **Flat o Git history do upstream** — preserve `upstream` remote. `git fetch upstream` deve funcionar. Nunca rebuild from scratch.
- **Editar migration upstream aplicada** — crie migration nova. Custom migrations usam numbering separado.
- **Fazer Wasender MCP uma dependency** — investigue antes de adotar.
- **Claimar "anti-ban guarantees"** — nunca. Teste por controle, não por promessa.
- **Docker destructive commands** em recursos não relacionados — nunca `docker system prune`, `docker volume prune`, `docker compose down` em recursos alheios. Nunca apague volumes sem confirmação.
- **Usar WAHA como transporte WhatsApp** — o projeto substitui WAHA por WasenderAPI (§5, §52.4). Não basta renomear env vars; exige compatibility analysis completa (matriz WAHA→Wasender).
- **Assumir comportamento do Wasender a partir do WAHA** — nunca. Documentação Wasender é autoritativa (§51.9). Inspecione autenticação, PAT, session key, sessões, QR, webhooks, incoming/outgoing, mídia, delivery, errors, rate limits, retries, session isolation antes de implementar.
- **Fazer deploy editando arquivos à mão na VPS** — deploy via Coolify a partir de GitHub (§46, §52.20, §53.20). Commit → push → PR → merge na `main` → CI publica imagem → VPS puxa. Nunca `build:`-only em prod, nunca editar `.env`/compose à mão para bump de versão.
- **Assumption de servidor fresco no deploy** — inspecione Docker/Coolify existente primeiro (§45, §52.21). Nunca toque recursos não relacionados. Preserve `upstream` remote.
- **Confundir três tipos de AI** — Deskcomm runtime AI ≠ external coding agent (Claude Code/OpenCode) ≠ future CRM AI (§52.1). Nenhuma dependência sobre a outra.
- **Falhas silenciosas** — workflows que não executam devem persistir estado `FAILED` com informações completas (§51.34).

### Obrigatório

- **Layer architecture (4 camadas não confundíveis):** CRM → Experiment Engine → Workflow Engine → Canonical WhatsApp Interface → Wasender Adapter → Wasender API
- **Arquitetura de sessão:** múltiplas sessões WhatsApp suportadas. Cada workflow execution sabe sua session. Session A nunca envia por Session B.
- **Canonical event model:** normalize eventos do Wasender antes do workflow engine. Payloads específicos do provedor não vazam para o CRM.
- **Single conversation:** human + automated messages compartilham uma história cronológica. Não crie "automation conversation" separada.
- **Time mockável:** produção usa random; testes usam deterministic seed.
- **Follow-up content separado da workflow logic:** template de mensagem ≠ media asset ≠ workflow step.
- **API programática é first-class:** workflow creation via API produce o mesmo representation do visual builder.
- **`docs/UPSTREAM-MODIFICATIONS.md`:** registre cada arquivo upstream modificado (File, Upstream responsibility, Why modified, Our modification, Dependency on custom functionality, How difficult to rebase, Possible future extraction).
- **Final app in English:** o produto final deve ser em inglês, não português. O upstream DeskcommCRM usa pt-BR como padrão com suporte parcial a `es` (`IDIOMAS = ["pt-BR", "es"]` em `lib/i18n/idiomas.ts`); `en` não está no array e nunca teve tradução. O custom layer deve: (1) adicionar `en` ao array `IDIOMAS`, (2) garantir tradução completa para inglês de todas as UI strings (a chave do dicionário é pt-BR, então `en` deve cobrir todas as 1.239+ entradas), (3) considerar `en` como idioma padrão para novos tenants, (4) manter `CLAUDE.md`/`AGENTS.md` em inglês, (5) usar inglês para variáveis/funções/comentários em código novo.
- **Version matrix:** Deskcomm version + Custom version + Wasender adapter + DB schema + Status.
- **Rebase test é release gate:** prove que o custom layer reaplica sobre novo upstream sem rebuild.
- **Test layers (todas obrigatórias):** Unit → Property-based → Integration → Fake Wasender → State-machine → Database → Concurrency/idempotency → UI E2E → Real Wasender → Real WhatsApp.
- **Two-number real WhatsApp test:** apenas após todos os testes simulados passarem. 2 números controlados.
- **Database integrity testing:** inspecione estado persistido, não apenas UI assertions.
- **Multi-tenant security testing:** Tenant A nunca vê dados de Tenant B. Add isolation tests.
- **Auditar repositório completo antes de codificar** — inspecione source code (não só README). Mapeie: WAHA integration, Supabase schema + RLS, event_log + workers, CRM/inbox/pipelines, automação existente, testes, Docker, env vars (§2, §51.6).
- **Contract-first Wasender** — crie interface explícita (`WhatsAppProvider` com `sendText/sendImage/sendAudio/sendVideo/sendDocument/getSessionStatus/handleWebhook/normalizeEvent`) antes de implementar (§51.8, §10).
- **Kill switches** — global automation pause, per-variant PAUSED, per-session automation pause. Cada um previne novos envios sem deletar dados históricos (§51.37-51.39).
- **Observabilidade** — logs estruturados, correlation IDs, workflow execution traceable de customer → message → event → variant → workflow → node → provider request/response (§51.33).
- **Domínio público** — necessário para webhook URLs (WasenderAPI calls back), app URLs, sessão WhatsApp. Sem domínio público HTTPS, webhooks não funcionam.
- **Checkpoint de Git** — após cada fase: implement + test + verify (§51.4). Ex: `v1.0-baseline`, `v1.1-wasender-adapter`, `v1.2-webhooks`, etc.
- **Golden baseline** — antes de modificar: deploy original + testes + walkthrough UI + config Docker/Coolify (§51.5). Regression-test contra isso após cada fase.
- **Análise de impacto** — antes de modificar componente upstream, gere: Current responsibility / Why required / Files affected / Database affected / API affected / Potential regression / How tested / Rollback (§51.3).

### Phase gates (obrigatório)

```
ANALYZE → MINIMAL IMPLEMENTATION → UNIT TEST → INTEGRATION TEST → E2E TEST
  → UI QA → REGRESSION → DOCUMENT → DEPLOY → OBSERVE → ONLY THEN ADD COMPLEXITY
```

V1 → V2 → V3 são progressivos. Nunca build V1+V2+V3 simultaneamente.

---

## Agent skills

### Issue tracker

Issues live on GitHub Issues for `pixarusemperor/wadeskhybrid`. See `docs/agents/issue-tracker.md`.

### Triage labels

Default triage vocabulary (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`, `wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `CONTEXT.md` + `docs/adr/` at the repo root. See `docs/agents/domain.md`.
