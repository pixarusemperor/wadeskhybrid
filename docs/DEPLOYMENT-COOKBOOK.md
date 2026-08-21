# Deployment Cookbook — GitHub ↔ Coolify

> **Single source of truth** for deploying any project from GitHub to Coolify.
> Written so any agent (or human) can set up a new project from scratch without
> re-discovering credentials, endpoints, or gotchas.

---

## Table of Contents

1. [Infrastructure Overview](#1-infrastructure-overview)
2. [Coolify Credentials](#2-coolify-credentials)
3. [GitHub Secrets](#3-github-secrets)
4. [Deploy Workflow — How It Works](#4-deploy-workflow--how-it-works)
5. [New Project Checklist (Step by Step)](#5-new-project-checklist-step-by-step)
6. [Coolify API Reference](#6-coolify-api-reference)
7. [Known Gotchas & Fixes](#7-known-gotchas--fixes)
8. [Currently Deployed Apps](#8-currently-deployed-apps)
9. [DNS & Domain Setup](#9-dns--domain-setup)
10. [Troubleshooting Flowchart](#10-troubleshooting-flowchart)

---

## 1. Infrastructure Overview

```
VPS:            Ubuntu 24.04, IP 34.155.88.118
Coolify:        v4.1.2
Panel URL:      https://coolifyone.orizongroup.online
API Base:       https://coolifyone.orizongroup.online/api/v1
SSL:            SELF-SIGNED — every API call needs curl -k (or NODE_TLS_REJECT_UNAUTHORIZED=0)
Docker network: coolify
DNS provider:   LWS (ns23.lwsdns.com, ns24.lwsdns.com)
```

### Deployment flow

```
You push code → GitHub Actions triggers → calls Coolify API → Coolify pulls
from GitHub, builds Docker image, deploys container → Actions polls until
deploy succeeds or fails → commit gets green/red status check
```

Two mechanisms work together:
- **Coolify auto-deploy** (`is_auto_deploy_enabled: true`) — monitors the linked
  Git branch and triggers a build on every push.
- **GitHub Actions workflow** — explicitly triggers the deploy via Coolify API
  AND polls the deployment status, so the commit gets a visible pass/fail badge.

You don't need both, but using both gives you redundant triggers AND visibility.

---

## 2. Coolify Credentials

### Where to find them

| Credential | Where to find it |
|---|---|
| **API Token** | Coolify panel → **Settings → API Tokens** (create a new one with deploy permission) |
| **Project UUID** | Coolify panel → click the Project → UUID shown in the URL bar or sidebar |
| **Environment UUID** | Inside the Project → click the Environment → same location |
| **Server UUID** | Coolify panel → **Servers** → click the server → shown in sidebar |
| **App UUID** | Coolify panel → click the Application → UUID shown in URL bar or sidebar |

### API Token format

```
Bearer <token>
```

The token value stored in GitHub secrets should be the raw token string (e.g.
`8|abc123def456...`). The workflow adds the `Bearer` prefix.

**Token rotation**: If the token stops working, generate a new one in
Coolify → Settings → API Tokens and update the GitHub secret.

### Direct database query (fallback)

If you can't find UUIDs in the panel, SSH into the VPS and query:

```bash
sudo docker exec -i coolify-db psql -U coolify -d coolify \
  -c "select id, uuid, name, fqdn, git_repository from applications;"
```

---

## 3. GitHub Secrets

### Required secrets (set on the repo)

| Secret name | Value | Required? |
|---|---|---|
| `COOLIFY_API_TOKEN` | Coolify API token (e.g. `8|abc123...`) | ✅ Yes |
| `COOLIFY_APP_UUID` | Application UUID from Coolify | ✅ Yes |
| `COOLIFY_BASE_URL` | `https://coolifyone.orizongroup.online` | ⚪ Optional (has default) |
| `COOLIFY_WEBHOOK` | Per-app Deploy Webhook URL | ⚪ Optional (preferred if set) |

### How to set secrets

```bash
# Via GitHub CLI
gh secret set COOLIFY_API_TOKEN --repo pixarusemperor/YOUR_REPO --body "8|your_token_here"
gh secret set COOLIFY_APP_UUID --repo pixarusemperor/YOUR_REPO --body "your_app_uuid_here"
gh secret set COOLIFY_BASE_URL --repo pixarusemperor/YOUR_REPO --body "https://coolifyone.orizongroup.online"
```

Or via GitHub web UI:
1. Go to `https://github.com/pixarusemperor/YOUR_REPO/settings/secrets/actions`
2. Click **New repository secret**
3. Enter name and value, click **Add secret**

### Organization-level tip

If you have many repos, set `COOLIFY_API_TOKEN` at the GitHub **organization
level** (`pixarusemperor`) and grant access to all repos. You only manage one
copy of the token.

### How to verify secrets are set

```bash
gh secret list --repo pixarusemperor/YOUR_REPO
# Should show: COOLIFY_API_TOKEN, COOLIFY_APP_UUID, etc.
```

---

## 4. Deploy Workflow — How It Works

The workflow lives at `.github/workflows/deploy.yml`. Here's what it does:

### Step 1: Trigger Deploy

**Preferred path** — if `COOLIFY_WEBHOOK` is set:
```bash
curl -sk -X GET "${COOLIFY_WEBHOOK}" \
  --header "Authorization: Bearer ${COOLIFY_TOKEN}"
```

**Fallback path** — uses `POST /api/v1/deploy` with app UUID:
```bash
curl -sk -X POST \
  -H "Authorization: Bearer ${COOLIFY_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{"uuid": "APP_UUID"}' \
  "https://coolifyone.orizongroup.online/api/v1/deploy"
```

### Step 2: Poll for Status

Polls `GET /api/v1/deployments/applications/{uuid}` every 15 seconds:
- `finished` → success (exit 0, green check)
- `failed` → failure (exit 1, red X)
- `queued` / `building` / `deploying` → continue polling
- After 40 attempts (10 minutes) → timeout (exit 1)

### Concurrency

```yaml
concurrency:
  group: coolify-deploy-${{ github.ref }}
  cancel-in-progress: true
```

If you push again while a deploy is running, the old one is cancelled.

---

## 5. New Project Checklist (Step by Step)

### Prerequisites

- [ ] GitHub repo exists (public, or with deploy key configured)
- [ ] Repo has a `Dockerfile` at the root
- [ ] Domain DNS A record points to `34.155.88.118`
- [ ] Coolify API token is available

### Step 1: Create the app in Coolify

**Via API:**
```bash
curl -sk -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "project_uuid": "o3kjmy9tmrvviidlvlv842vk",
    "environment_uuid": "anwdebql2ddtufiotv3mcbe7",
    "server_uuid": "ypl10ghx88it0xefro9d3duf",
    "name": "your-app-name",
    "build_pack": "dockerfile",
    "git_repository": "https://github.com/pixarusemperor/your-repo.git",
    "git_branch": "main",
    "domains": "https://your-domain.orizongroup.online",
    "ports_exposes": "3000",
    "ports_mappings": "3000:3000",
    "base_directory": "/",
    "dockerfile_location": "/Dockerfile",
    "is_auto_deploy_enabled": true,
    "is_force_https_enabled": true
  }' \
  "https://coolifyone.orizongroup.online/api/v1/applications/public"
```

**CRITICAL**: Always include `dockerfile_location: "/Dockerfile"` and
`base_directory: "/"` in the creation payload. Without them, the build
uses wrong paths and fails.

**Via UI**: Coolify → New Resource → Application → Public Repository → fill in fields.

### Step 2: Get the App UUID

After creation, find the UUID:
- In the Coolify panel URL bar when viewing the app
- Or via API: `GET /api/v1/applications` (lists all apps with UUIDs)
- Or via database query on the VPS (see Section 2)

### Step 3: Set environment variables

**Via API** (one at a time — bulk endpoint is unreliable):
```bash
curl -sk -X POST \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key": "VAR_NAME", "value": "var_value"}' \
  "https://coolifyone.orizongroup.online/api/v1/applications/APP_UUID/envs"
```

**Via UI**: Coolify → App → Environment Variables → add each one.

**Build variables**: For `NEXT_PUBLIC_*` vars that Next.js inlines at build
time, toggle **Build Variable = ON** in the Coolify UI.

### Step 4: Set GitHub secrets

```bash
gh secret set COOLIFY_API_TOKEN --repo pixarusemperor/YOUR_REPO --body "YOUR_TOKEN"
gh secret set COOLIFY_APP_UUID --repo pixarusemperor/YOUR_REPO --body "APP_UUID_FROM_STEP_2"
```

### Step 5: Copy and customize the deploy workflow

```bash
cp ~/WAdeskhybrid/.github/workflows/deploy.yml \
   ~/your-project/.github/workflows/deploy.yml
```

Edit the workflow:
1. Update the comment header (project name)
2. Set `branches:` to your branch name
3. `COOLIFY_APP_UUID` is read from secrets — no code change needed

### Step 6: Push and verify

```bash
git push origin main
# Then check: https://github.com/pixarusemperor/YOUR_REPO/actions
```

The workflow should trigger, deploy, and report green within ~5 minutes.

---

## 6. Coolify API Reference

All endpoints require:
- Header: `Authorization: Bearer YOUR_TOKEN`
- SSL: self-signed (use `curl -k`)

### Applications

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `GET` | `/api/v1/applications/{uuid}` | Get app details and current status |
| `GET` | `/api/v1/applications` | List all applications |
| `POST` | `/api/v1/applications/public` | Create app from public repo |
| `PATCH` | `/api/v1/applications/{uuid}` | Update app config or trigger deploy |
| `DELETE` | `/api/v1/applications/{uuid}` | Delete an application |
| `POST` | `/api/v1/applications/{uuid}/envs` | Add environment variable |

### Deployments

| Method | Endpoint | Purpose |
|--------|----------|---------|
| `POST` | `/api/v1/deploy` | Trigger deploy (body: `{"uuid": "APP_UUID"}`) |
| `GET` | `/api/v1/deployments/applications/{uuid}` | List deployments with status |

### Common operations

**Trigger deploy (preferred — POST /api/v1/deploy):**
```bash
curl -sk -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"uuid": "APP_UUID"}' \
  "https://coolifyone.orizongroup.online/api/v1/deploy"
```

**Trigger deploy (alternative — PATCH with instant_deploy):**
```bash
curl -sk -X PATCH \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"instant_deploy": true}' \
  "https://coolifyone.orizongroup.online/api/v1/applications/APP_UUID"
```

**Check app status:**
```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://coolifyone.orizongroup.online/api/v1/applications/APP_UUID"
```

**Check deployment logs:**
```bash
curl -sk -H "Authorization: Bearer $TOKEN" \
  "https://coolifyone.orizongroup.online/api/v1/deployments/applications/APP_UUID"
```

**Add env var:**
```bash
curl -sk -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"key": "VAR_NAME", "value": "var_value"}' \
  "https://coolifyone.orizongroup.online/api/v1/applications/APP_UUID/envs"
```

### Deployment status values

| Status | Meaning |
|--------|---------|
| `queued` | Waiting for a build slot |
| `building` | Docker build in progress |
| `deploying` | Container being started |
| `finished` | Deploy succeeded |
| `failed` | Build or deploy failed |

---

## 7. Known Gotchas & Fixes

### SSL Certificate (self-signed)

```
curl: (60) SSL certificate problem: self-signed certificate
```

**Fix**: Always use `curl -k`. For Node.js: `NODE_TLS_REJECT_UNAUTHORIZED=0`.

### Port Already in Use

```
failed to bind host port 0.0.0.0:3000/tcp: address already in use
```

**Fix**: Find and kill the competing process:
```bash
sudo lsof -i :3000
kill -9 <PID>
```

### API Validation Error

```
"This field is not allowed"
```

**Fix**: Add environment variables **one at a time** using
`POST /api/v1/applications/{uuid}/envs`. The bulk endpoint is unreliable.

### Dockerfile Location Not Set

**Symptom**: Build fails or uses wrong Dockerfile path.

**Fix**: Always include in creation payload:
```json
"dockerfile_location": "/Dockerfile",
"base_directory": "/"
```

### Traefik Routing Lag (30–60 seconds)

**Symptom**: Container is healthy but domain returns 404/502.

**Fix**: Wait 60 seconds. If still broken, the static Traefik config may need
updating. Check `/traefik/dynamic/coolify.yaml` inside the `coolify-proxy`
container.

**WARNING**: Static Traefik routes point to a specific container IP. On redeploy,
the container IP changes. Wait for Docker provider sync (30–60s) first.

### Multiline Environment Variables

**Symptom**: Docker build fails with syntax errors.

**Fix**: Base64-encode multiline values and decode in the entrypoint script:
```bash
echo "${VAR_B64}" | base64 -d > /tmp/decoded.txt
```

### `--fail-with-body` Not Available

**Symptom**: `curl: unknown option: --fail-with-body`

**Fix**: GitHub Actions runners have curl 7.81+ (supports it). For local
testing with older curl, use `--fail` instead.

### Deploy Webhook vs API Token

The workflow tries two methods:
1. **Deploy Webhook** (preferred) — a GET request triggers deploy, no UUID needed
2. **POST /api/v1/deploy** (fallback) — needs both token and app UUID

If you have a `COOLIFY_WEBHOOK` secret, the workflow uses method 1. Otherwise
it falls back to method 2.

### Private Repos

Coolify needs a deploy key for private repos. Options:
1. Make the repo public (simplest)
2. Add an SSH deploy key in Coolify → Settings → Keys & Tokens
3. Use a GitHub PAT with repo scope

All current projects use **public repos** (simplest path).

---

## 8. Currently Deployed Apps

| App | Domain | App UUID | GitHub Repo | Branch |
|-----|--------|----------|-------------|--------|
| wadeskhybrid | `wassflow.orizongroup.online` | `mbnoymz1gltpvx2dl3ubdrz2` | `pixarusemperor/wadeskhybrid` | `main` |
| WassFlow SaaS | _(stopped, domain removed)_ | `zxt32b72sbm7bsixg1s2rhr8` | `pixarusemperor/whatsapp-chatbot-saas` | `main` |
| wacrm-wasender | `wasender.orizongroup.online` | `jrd07b6d5zn18kr0i8y7bz16` | — | — |
| Image Ads Generator | `superads.orizongroup.online` | `bwng78yv21ngxycdcnbbput8` | `pixarusemperor/whatsapp-chatbot-saas` | `master` |

**Infrastructure UUIDs** (same for all apps on this Coolify instance):
```
Project UUID:     o3kjmy9tmrvviidlvlv842vk
Environment UUID: anwdebql2ddtufiotv3mcbe7
Server UUID:      ypl10ghx88it0xefro9d3duf
Destination UUID: ci60o5j5af5hcgilo2otx98m
```

### To deactivate an app

**Via API:**
```bash
curl -sk -X PATCH \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"is_paused": true}' \
  "https://coolifyone.orizongroup.online/api/v1/applications/APP_UUID"
```

**Via UI**: Coolify → App → Settings → Pause/Stop.

**To delete entirely:**
```bash
curl -sk -X DELETE \
  -H "Authorization: Bearer $TOKEN" \
  "https://coolifyone.orizongroup.online/api/v1/applications/APP_UUID"
```

---

## 9. DNS & Domain Setup

### Adding a new subdomain

1. Go to your DNS provider (LWS panel)
2. Create an **A record**:
   - Host: `your-subdomain`
   - Value: `34.155.88.118`
   - TTL: 300 (5 minutes)
3. Wait for propagation (~5 min)
4. Set the Coolify app domain to `https://your-subdomain.orizongroup.online`

### Existing subdomains

| Subdomain | Points to | Status |
|---|---|---|
| `wassflow.orizongroup.online` | 34.155.88.118 | Active (WassFlow SaaS) |
| `superads.orizongroup.online` | 34.155.88.118 | Active (Image Ads) |

---

## 10. Troubleshooting Flowchart

```
Deploy failed?
│
├─ GitHub Actions shows red?
│  ├─ "Missing COOLIFY_API_TOKEN" → set the secret (Section 3)
│  ├─ "No deployment UUID returned" → token lacks deploy permission → regenerate in Coolify
│  └─ "TIMEOUT" → check Coolify panel for build logs
│
├─ Deploy succeeded but domain returns 404/502?
│  ├─ Wait 60 seconds (Traefik sync lag)
│  ├─ Still broken? Check container is running in Coolify panel
│  └─ Still broken? Check Traefik config in coolify-proxy container
│
├─ Deploy succeeded but app crashes on boot?
│  ├─ Check env vars in Coolify panel (missing or wrong?)
│  ├─ Check Coolify deployment logs for stack traces
│  └─ Ensure NEXT_PUBLIC_* vars have "Build Variable" toggle ON
│
├─ "Port already in use" during build?
│  └─ Kill competing process on port 3000 (lsof -i :3000 → kill -9)
│
└─ SSL/certificate errors?
   └─ Always use curl -k for Coolify API calls
```

---

*Last updated: 2026-08-21*
*Maintained by: wadeskhybrid project*
