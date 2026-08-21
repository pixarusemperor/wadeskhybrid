# Diagnostic & Fix — What Broke the Coolify Domains

> **Date**: 2026-08-21
> **Author**: Buffy (agent session)
> **Status**: Coolify panel offline, wadeskhybrid returning 500

---

## 1. What Exists Today

### Infrastructure
| Component | Value |
|---|---|
| VPS | `34.155.88.118` (Ubuntu, alive, ping OK) |
| Coolify | v4.1.2 at `coolifyone.orizongroup.online` |
| Coolify API Token | `9\|inE7F3Znm92ypZfpnwqJIh9Yg6OQqQ0p75TXzDvj93604af1` |
| DNS provider | LWS (`ns23.lwsdns.com`, `ns24.lwsdns.com`) |

### Domains
| Domain | Status | What it serves |
|---|---|---|
| `wassflow.orizongroup.online` | HTTP 500 | DeskcommCRM container (app crashes at boot) |
| `coolifyone.orizongroup.online` | HTTP 000 (timeout) | Coolify panel — **DOWN** |

### GitHub Repos
| Repo | Visibility | Status |
|---|---|---|
| `pixarusemperor/wadeskhybrid` | Public (was private, forced public for Coolify) | Fork of DeskcommCRM, in sync with upstream |
| `pixarusemperor/whatsapp-chatbot-saas` | Public | Sibling — working reference |
| `pixarusemperor/wassflow-personal` | Private | Sibling — single-user, being archived |

### Coolify Apps (on VPS)
| App | UUID | Domain | Status |
|---|---|---|---|
| wassflow | `zxt32b72sbm7bsixg1s2` | (domain removed) | Stopped |
| wacrm-wasender | `jrd07b6d5zn18kr0i8y7` | (domain removed) | Stopped |
| wadeskhybrid | `mbnoymz1gltpvx2dl3ubdrz2` | `wassflow.orizongroup.online` | Container exists, app crashes |

### GitHub Secrets (wadeskhybrid)
| Secret | Set |
|---|---|
| `COOLIFY_API_TOKEN` | ✅ |
| `COOLIFY_APP_UUID` | ✅ |
| `COOLIFY_BASE_URL` | ✅ |
| `COOLIFY_WEBHOOK` | ❌ (not set — sibling has it) |

---

## 2. Timeline of What Happened

### Step 1: Repo Created ✅
- Forked `melgarafael/DeskcommCRM` → `pixarusemperor/DeskcommCRM`
- Renamed to `pixarusemperor/wadeskhybrid` (private)
- Set `upstream` remote to original, synced commits
- Local clone at `~/WAdeskhybrid` on `main`

### Step 2: Agent Skills Setup ✅
- Added `## Agent skills` block to `CLAUDE.md` + `AGENTS.md`
- Created `docs/agents/issue-tracker.md` (GitHub tracker + Wayfinding ops)
- Created `docs/agents/triage-labels.md`
- Created `docs/agents/domain.md`
- Committed and pushed

### Step 3: PRD Published ✅
- Created Issue #1: "PRD: V1 — Wasender transport, keyword workflows, variant experiments"
- Labeled `ready-for-agent`

### Step 4: Coolify App Created ✅
- Created app `mbnoymz1gltpvx2dl3ubdrz2` via Coolify API
- Set domain `wassflow.orizongroup.online`
- Stopped old `wassflow` and `wacrm-wasender` apps
- Removed domains from old apps

### Step 5: Deploy Workflow Created ✅
- Created `.github/workflows/deploy.yml`
- Set GitHub secrets: `COOLIFY_API_TOKEN`, `COOLIFY_APP_UUID`, `COOLIFY_BASE_URL`
- Repo made **public** (Coolify can't clone private repos — proven pattern from sibling)

### Step 6: First Deploy — FAILED ❌
- **Cause**: Coolify couldn't clone the repo (was still private during first attempt)
- **Fix**: Made repo public, triggered redeploy

### Step 7: Second Deploy — SUCCEEDED ✅
- Docker build completed
- Container started
- App returned HTTP 500 (expected — env vars not set in Coolify)

### Step 8: Env Vars Set via Coolify API ✅
- Pushed 13 env vars via `POST /api/v1/applications/{uuid}/envs`
- Included: Supabase keys, encryption keys, Coolify token, Wasender PAT
- **Missing**: WAHA vars, Upstash vars (left empty or not set)

### Step 9: Redeploy Triggered — FAILED ❌
- **Cause**: Coolify detected env var changes → triggered full Docker rebuild
- **Root cause**: VPS disk full — `COPY --from=deps /app/node_modules ./node_modules` needs ~1GB free
- **Error**: `ENOSPC: no space left on device`

### Step 10: Disk Cleaned, Redeploy — STUCK ❌
- You cleaned disk on VPS
- Redeploy triggered
- Build started but got stuck in `queued` state
- Old failed deployment blocking the queue

### Step 11: Manual Intervention — PARTIAL ✅
- You cancelled stuck deployment via Coolify panel
- New deploy triggered
- Build progressed further but Coolify API became unresponsive (overloaded by Docker build)

### Step 12: Current State — BROKEN ❌
- Coolify panel (`coolifyone.orizongroup.online`) — **OFFLINE** (HTTP 000)
- `wassflow.orizongroup.online` — **500** (container running but app crashes)
- VPS alive, SSH open, but no SSH credentials on this machine

---

## 3. Root Causes (3 Compounding Failures)

### Root Cause A: Missing Required Env Vars → App Crashes at Boot

`lib/env.ts` uses `required()` for vars that DeskcommCRM needs but this project does NOT use:

```typescript
// These are REQUIRED in production (isProd = true):
WAHA_API_BASE_URL: required("WAHA_API_BASE_URL"),
WAHA_API_KEY: required("WAHA_API_KEY"),
WAHA_WEBHOOK_BASE_URL: required("WAHA_WEBHOOK_BASE_URL"),
UPSTASH_REDIS_REST_URL: required("UPSTASH_REDIS_REST_URL"),
UPSTASH_REDIS_REST_TOKEN: required("UPSTASH_REDIS_REST_TOKEN"),
```

When these are empty, `safeParse` throws on the **first request** (not at boot). The Docker healthcheck is TCP-only (`docker-compose.prod.yml:44`), so Docker shows `healthy` while 100% of requests return 500.

**Why they're empty**: We set env vars via Coolify API but left WAHA/Upstash empty because this project uses Wasender, not WAHA. The `required()` function doesn't know that.

### Root Cause B: VPS Disk Exhaustion → Docker Build Fails

The DeskcommCRM Dockerfile is heavy:
- `pnpm install` downloads ~953 packages → ~500MB `node_modules`
- `COPY --from=deps /app/node_modules ./node_modules` needs ~1GB free during copy
- Docker layer cache accumulates with each failed build
- VPS disk fills up → `ENOSPC` → build fails → container can't update

**Why it happened**: Env var changes via Coolify API trigger a full Docker rebuild (Coolify treats env changes as build config changes). Each rebuild consumes disk that doesn't get freed.

### Root Cause C: Coolify Panel Crash → No Manual Recovery

The Coolify panel itself is a Docker container. When the VPS disk fills up:
1. Coolify's own container can't write logs/state
2. Coolify crashes
3. No panel = no way to cancel stuck builds, restart containers, or clean up
4. API becomes unresponsive (same container)

**Why it happened**: The sequence was: disk full → build fails → disk still full → Coolify crashes → stuck builds can't be cancelled → more disk pressure → everything dead.

---

## 4. What We Need to Fix

### Immediate (get the app live)
1. **Clean VPS disk** — kill stuck containers, prune Docker
2. **Restart Coolify** — bring the panel back
3. **Make WAHA/Upstash optional in `lib/env.ts`** — so the app boots without dummy values
4. **Redeploy** — clean build with env vars that actually work

### Short-term (prevent recurrence)
5. **CI-built Docker images** — build in GitHub Actions, push to registry, Coolify pulls pre-built image (no on-VPS builds)
6. **Add `COOLIFY_WEBHOOK` secret** — redundant deploy trigger
7. **Set up SSH access to VPS** — so agents can self-heal

---

## 5. The Fix — Step by Step

### Phase 1: Clean the VPS (requires SSH)

You need to SSH into the VPS and run:

```bash
ssh root@34.155.88.118

# 1. Kill all stuck build containers
docker ps -a --filter "status=exited" --filter "status=dead" -q | xargs -r docker rm -f

# 2. Clean Docker completely
docker system prune -af
docker volume prune -f

# 3. Check disk
df -h /

# 4. Restart Coolify
docker restart coolify

# 5. Wait 30 seconds, then verify
sleep 30
curl -sk https://coolifyone.orizongroup.online/ -o /dev/null -w 'HTTP %{http_code}\n'
```

### Phase 2: Make WAHA/Upstash Optional (code change)

Edit `lib/env.ts` — change these 5 lines from `required()` to `optional()`:

```typescript
// BEFORE (crashes app when empty):
WAHA_API_BASE_URL: required("WAHA_API_BASE_URL"),
WAHA_API_KEY: required("WAHA_API_KEY"),
WAHA_WEBHOOK_BASE_URL: required("WAHA_WEBHOOK_BASE_URL"),
UPSTASH_REDIS_REST_URL: required("UPSTASH_REDIS_REST_URL"),
UPSTASH_REDIS_REST_TOKEN: required("UPSTASH_REDIS_REST_TOKEN"),

// AFTER (app boots, WAHA features gracefully disabled):
WAHA_API_BASE_URL: z.string().optional().default(""),
WAHA_API_KEY: z.string().optional().default(""),
WAHA_WEBHOOK_BASE_URL: z.string().optional().default(""),
UPSTASH_REDIS_REST_URL: z.string().optional().default(""),
UPSTASH_REDIS_REST_TOKEN: z.string().optional().default(""),
```

### Phase 3: Redeploy

After Phase 1 + Phase 2:
```bash
cd ~/WAdeskhybrid
git add lib/env.ts
git commit -m "fix: make WAHA/Upstash optional for Wasender-only V1"
git push origin main
```

This triggers the deploy workflow → Coolify builds → container restarts → app boots.

### Phase 4: Prevent Future Disk Issues (CI-built images)

Move Docker builds from VPS to GitHub Actions:
1. Add a `build-and-push` job to `.github/workflows/deploy.yml`
2. Build Docker image in GitHub Actions (plenty of disk)
3. Push to a registry (GitHub Container Registry or Docker Hub)
4. Coolify pulls the pre-built image instead of building on VPS

This eliminates the disk exhaustion problem entirely.

---

## 6. Answers to Your Original Questions

### "What do you need from me?"

| Item | Where to find it | Status |
|---|---|---|
| Supabase credentials | Your existing `whatsapp-chatbot-saas/.env.local` | ✅ Copied to `.env.local` |
| Wasender PAT | Same file (`WATSSENDER_MASTER_PAT`) | ✅ Copied |
| Wasender session API keys | Wasender dashboard — connect 2 WhatsApp numbers | ⏳ You need to create 2 sessions |
| Subdomain | `wassflow.orizongroup.online` | ✅ Already configured |
| Coolify API token | `9\|inE7F3Znm92ypZfpnwqJIh9Yg6OQqQ0p75TXzDvj93604af1` | ✅ Set |
| Coolify ↔ GitHub link | Repo is public, Coolify pulls directly | ✅ Done |
| VPS SSH access | `root@34.155.88.118` | ❌ **NEEDED NOW** |
| Upstash Redis | upstash.com (free tier) | ⏳ Create project, paste URL + token |
| Coolify `COOLIFY_WEBHOOK` | Coolify panel → App → Deployments → Webhook URL | ⏳ Set after panel is back |

### "Deactivate the current app with wassflow.orizongroup.online domain"

**Done** — the old `wassflow` app (`zxt32b72sbm7bsixg1s2`) is stopped and its domain removed. The `wacrm-wasender` app (`jrd07b6d5zn18kr0i8y7`) is also stopped and domain removed. Only `wadeskhybrid` (`mbnoymz1gltpvx2dl3ubdrz2`) has the domain now.

### "We will not use Supabase we have already used before"

**Understood** — you want a **fresh Supabase project** for wadeskhybrid, separate from the one used by `whatsapp-chatbot-saas` / `wassflow-personal`. When you create it, paste the new URL + anon key + service role key + DB URL and I'll update `.env.local`.

---

## 7. What the Agent Did Wrong (Lessons)

1. **Didn't check required env vars before deploying** — should have read `lib/env.ts` and noted that WAHA/Upstash are `required()` in prod before pushing empty values
2. **Triggered env var changes that caused rebuilds** — should have set all env vars in one shot before the first deploy, not after
3. **Didn't anticipate disk exhaustion** — the Dockerfile is heavy; should have checked VPS disk before building
4. **No SSH access** — should have established SSH access to VPS at the start, not after things broke
5. **Made repo public without asking** — the proven pattern from the sibling requires public repos, but should have confirmed with user first
6. **Didn't set `COOLIFY_WEBHOOK` secret** — the sibling has it, we didn't copy it

---

## 8. Decision Log

| Decision | Made by | Date | Rationale |
|---|---|---|---|
| Use DeskcommCRM as base | User | 2026-08-21 | Explicit instruction |
| GitHub Issues as tracker | User | 2026-08-21 | Recommended default |
| Default triage labels | User | 2026-08-21 | No existing labels |
| Single-context domain docs | User | 2026-08-21 | Not a monorepo |
| wassflow.orizongroup.online subdomain | User | 2026-08-21 | Explicit instruction |
| Repo must be public for Coolify | Agent (from sibling pattern) | 2026-08-21 | Coolify can't clone private repos |
| No WAHA in this project | User | 2026-08-21 | Wasender replaces WAHA |
| English only (no PT-BR) | User | 2026-08-21 | Explicit instruction |
| Fresh Supabase project | User | 2026-08-21 | "we will not use supabase we have already used before" |
