# Fybud Deploy — make a repo shippable

Drop this file at the **repo root**. It is the only file Fybud org automation seeds into new repos.
Follow it to add GitHub Actions + `docker-compose.deploy.yml` so push-to-`main` goes live on
`https://{tool}.fybud.com`.

Pipeline (no Coolify / CapRover / Traefik):

`git push main` → GitHub Actions builds/pushes `fybud/*` images → Fybud Deploy webhook →
downloads only root `docker-compose.deploy.yml` (no repo clone) → free host ports + `.env` →
`docker compose pull && up` → Cloudflare A if missing → host nginx → TLS.

First deploy: paste secrets in Deploy UI + approve. Later pushes auto-redeploy.

## Deployment safety contract

A deployment is a transaction around a currently live release. A failure at any
stage must leave that release serving, or restore the exact last known-good
release. The control plane should report a deployment as successful only after
all of these stages complete:

```
accepted -> compose validated -> image pulled -> candidate running
-> host health gate -> DNS ready -> TLS ready -> nginx tested and reloaded
-> public HTTPS verified -> successful
```

For the control-plane implementation, the non-negotiable rules are:

- Serialize deploys per project and serialize global mutations (port allocation,
  DNS ownership, nginx write/test/reload). A webhook delivery is not a lock.
- Treat `(repo, commit SHA)` as idempotent. Ignore duplicate deliveries and
  superseded commits; fetch the compose file at the same immutable commit as the
  image tag, never from a moving `main` ref.
- Validate the downloaded compose and its policy before `pull` or `up`. Reject
  privileged services, host networking/PID/IPC, Docker socket or arbitrary host
  binds, non-loopback published ports, unsupported labels/networks, mutable image
  tags, and services without CPU/memory limits or `restart: unless-stopped`.
- Pull and verify every candidate image before changing containers. Retain the
  local current and previous image; do not depend on a registry pull during
  rollback.
- Use an atomic nginx update: write a candidate config, test the complete nginx
  configuration, atomically promote it, then reload. If test or reload fails,
  restore the previous site file and leave the loaded configuration untouched.
- Persist the phase, image digest, compose commit, allocated ports, and encrypted
  environment snapshot on every run. Restart recovery reconciles that desired
  state with Docker, nginx, DNS, and TLS rather than blindly replaying a job.
- A database migration is not automatically rollback-safe. Use backward-compatible
  expand/migrate/contract changes and run destructive migrations only in a later
  release.

`fybud.health` is liveness/readiness for an exposed service, not proof that an
application workflow works. Add an optional, app-specific smoke check before
promotion when `/health` alone is insufficient.

---

## 0. Repo layout (required)

```
MyTool/
  DEPLOY.md                         ← this file
  AGENTS.md                         ← agent contract (copy from Fybud workspace)
  docker-compose.deploy.yml         ← ONLY compose Deploy reads (repo root)
  .github/workflows/build-push.yml  ← build → Hub → Deploy webhook
  api/  web/  worker/ …             ← app code as folders, not separate repos
```

| File | Role |
|---|---|
| `docker-compose.deploy.yml` | **At repo root only** (not `.yaml`). Deploy downloads this single file via GitHub API — no git clone. |
| `.github/workflows/build-push.yml` | Builds images, pushes to Docker Hub `fybud/*`, POSTs the Deploy webhook. |
| `DEPLOY.md` | This playbook. |

Do **not** put the deploy compose under `infra/`. Do **not** create `infra/{tenant}/`, hand-written
nginx, or per-app Postgres containers.

---

## 1. Create `docker-compose.deploy.yml` (repo root)

Image-only (CI already built the images). Public services need labels + loopback host ports.
Private workers/redis: no `ports:`, no `fybud.expose` / `fybud.domain`.

```yaml
name: mytool

services:
  api:
    image: fybud/mytool-api:${IMAGE_TAG:?IMAGE_TAG required}
    restart: unless-stopped
    env_file: [.env]
    environment:
      PORT: 4100                                            # literal — fixed in compose
      JWT_SECRET:                                           # empty — PASTE in Deploy UI
      STRIPE_SECRET_KEY:                                    # empty — PASTE (every secret the app reads)
      DATABASE_URL:                                         # empty — PASTE in Deploy UI
      CORS_ORIGIN:                                          # empty — PASTE https://mytool.fybud.com
      PUBLIC_API_URL:                                       # empty — Deploy-injected from fybud.domain
    ports:
      - "127.0.0.1:${API_HOST_PORT}:4100"                   # HOST port filled by Deploy
    labels:
      fybud.expose: "true"
      fybud.domain: api.mytool.fybud.com
      fybud.role: api
      fybud.health: "/health"                               # strict — must match healthcheck path
    healthcheck:
      # Prefer Node/Python over wget — many slim images do not ship wget/curl.
      test: ["CMD-SHELL", "node -e \"fetch('http://127.0.0.1:4100/health').then(r=>process.exit(r.status<400?0:1)).catch(()=>process.exit(1))\""]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 20s
    networks: [fybud-net]

  web:
    image: fybud/mytool-web:${IMAGE_TAG:?IMAGE_TAG required}
    restart: unless-stopped
    env_file: [.env]
    environment:
      # Deploy injects https://api.mytool.fybud.com from fybud.domain — never paste.
      VITE_API_BASE_URL:
    ports:
      - "127.0.0.1:${WEB_HOST_PORT}:5173"
    labels:
      fybud.expose: "true"
      fybud.domain: mytool.fybud.com
      fybud.role: web
      fybud.health: "any"                                   # SPA / static — process-up only
    networks: [internal]

  worker:                       # private — no ports, no expose labels
    image: fybud/mytool-api:${IMAGE_TAG:?IMAGE_TAG required}
    restart: unless-stopped
    command: ["worker"]
    env_file: [.env]
    environment:
      # Declare every key the worker reads (empty = paste, or ${:?} = injected).
      JWT_SECRET:
      DATABASE_URL:
    networks: [fybud-net, internal]

networks:
  fybud-net: { name: fybud-net, external: true }   # only DB-facing services
  internal: { driver: bridge }
```

### Labels (only these)

| Label | When | Meaning |
|---|---|---|
| `fybud.expose: "true"` | Public only | Publish + DNS + nginx |
| `fybud.domain` | Required if expose | FQDN e.g. `cep.fybud.com` |
| `fybud.health` | **Required** if expose | How Deploy probes the host port after `compose up` |
| `fybud.role` | Optional | `web` / `api` — disambiguates domains for env computation |

`fybud.health` values:

| Value | Behaviour | Use when |
|---|---|---|
| `"/health"` (or `"/api/health"`) | **Strict**: host probe requires HTTP &lt; 400 (4xx/5xx → fail → auto-rollback) | Public **API** with a real health route |
| `"any"` | Process-up: any HTTP response on that port counts | **Web/SPA/static** (no health route), or APIs that auth-guard every path |
| *(absent)* | Legacy lenient probe of `/` — Deploy warns you to declare the label | Never for new services |

Rules:

- Every **public API** declares a compose `healthcheck:` **and** a matching strict `fybud.health` path.
- Every **public web** service uses `fybud.health: "any"` unless it has a dedicated health route.
- Deploy’s gate requires containers **running**, then probes `127.0.0.1:<hostPort>` using `fybud.health`. Docker’s own `healthy`/`starting` status is **not** a hard fail (images often lack wget).
- Do not invent health endpoints just for Deploy — use `"any"` instead.

### Env: declare every variable under `environment:`

`env_file: [.env]` loads values but does **not** say which keys must exist. Every variable a
service reads must also appear under `environment:` — **literally all of them**, including every
secret in the tool’s `requiredEnv` (Deploy Settings → Tool specs / `tools.ts`). There is **no
optional** form — only:

| Value in `environment:` | Meaning |
|---|---|
| `PORT: 4100` | **Literal** — value is fixed in the compose file |
| `JWT_SECRET:` (empty) | **Paste** in the Deploy UI — empty value *is* the flag |
| `VITE_API_BASE_URL:` / `PUBLIC_API_URL:` (empty) | **Deploy-injected** from `fybud.domain` — declare empty, never paste, never put in `requiredEnv` |
| `${VAR:?message}` | **Deploy-injected** (`*_HOST_PORT`, `IMAGE_TAG`) — never pasted |

Deploy injects these public URL keys from compose `fybud.domain` labels (do not hardcode or paste):

| Key | Value |
|---|---|
| `VITE_API_BASE_URL` | `https://` + API host |
| `PUBLIC_API_URL` | same as API host (backend public URL) |
| `PLATFORM_API_BASE_URL` | same as API host |
| `PLATFORM_WEB_BASE_URL` | `https://` + web host |
| `VITE_ADMIN_API_BASE_URL` / `PLATFORM_ADMIN_WEB_BASE_URL` | when admin hosts exist |

Do **not** use `${KEY:-}`, `${KEY:-default}`, or an “optionalEnv” list. If the app needs a key,
either give it a literal in compose, leave it empty to paste, or leave it empty for a Deploy-injected name.

Cross-check before merge:

1. Every `requiredEnv` key appears as `KEY:` (empty) under some service’s `environment:`.
2. Every empty paste `KEY:` in compose is listed in that tool’s `requiredEnv`, including every `*DATABASE*_URL` — **except** Deploy-injected public URL keys above.
3. Host ports / `IMAGE_TAG` use `${KEY:?…}` (never empty, never `${KEY:-}`).

Deploy refuses approve / env-save without every **paste** key (`missing[]`). Host ports,
`IMAGE_TAG`, and public URL injects are never pasted. Paste every declared `*DATABASE*_URL`
using the shared Postgres URL format below.

### SPA / Vite: runtime `env.js` (required for public frontends)

Vite bakes `import.meta.env.VITE_*` at **image build** time. CI must **not** bake production API
hosts into the image. Instead:

1. Compose declares `VITE_API_BASE_URL:` (empty) on the web service — Deploy injects
   `https://api.{tool}.fybud.com` into the container env.
2. The web image’s entrypoint writes `/env.js` from `VITE_*` at container start.
3. `index.html` loads `/env.js` **before** the app bundle; the SPA reads
   `window.__ENV__.VITE_API_BASE_URL` (fallback to `import.meta.env` for local dev).

If `/env.js` fails to load in the browser (`ERR_NAME_NOT_RESOLVED` / 404), login shows
**Failed to fetch** and may hit same-origin `/api/...` on the web host (wrong — API is on
`api.{tool}.fybud.com`). Fix DNS / hard-refresh; confirm `https://{tool}.fybud.com/env.js`
shows the API URL.

Also paste `CORS_ORIGIN=https://{tool}.fybud.com` (or your real web origin) on the API when
the backend enforces CORS.

### Ports / network / DB

- Ports: always `127.0.0.1:${*_HOST_PORT}:<container>` — never hardcode host ports.
- Shared Postgres hostname: `fybud-postgres` on external `fybud-net`. Paste each DB URL in the
  Deploy UI; Deploy validates it and **creates the database if it does not exist** before `docker
  compose up`.
- Shared Postgres login (superuser / admin): **username `postgres`, password `postgres`**.
  Example: `postgresql://postgres:postgres@fybud-postgres:5432/<db_name>`. The database name
  need not match the tool slug and multiple projects may share a database. See the **Databases**
  sidebar for mappings.
- Before compose, Deploy also checks for conflicting `container_name:` values and domains.
  Fix the compose / env and redeploy — you will get a toast with the conflict.
- Disk writers need a named volume (containers are recreated every deploy).
- Domains: web `{tool}.fybud.com`, API `api.{tool}.fybud.com`.
- After renaming `fybud.domain` labels, Deploy expands the Let’s Encrypt SAN list
  (DNS-01 via Cloudflare). If ssl: logs still say `skipping certbot` without `covers` /
  `expanding`, rebuild deploy-api on the VPS (`docker compose up -d --build api`) — tool
  redeploys do not update the control plane.
- Multi-tenant **admin** portals still need their own control-plane DB URL (e.g.
  `ADMIN_DATABASE_URL` → `cep-admin`) for the org registry. Per-client CEP DBs are pasted
  inside the admin UI, not as Deploy `requiredEnv` for every client.

---

## 2. Create `.github/workflows/build-push.yml`

Builds each service image, pushes `fybud/<service>:sha-…` + `:latest`, then notifies Deploy.

```yaml
name: build-push

on:
  push:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: build-push-${{ github.ref }}
  cancel-in-progress: true

env:
  DOCKERHUB_ORG: fybud
  TOOL: mytool          # Deploy tool slug

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.tag.outputs.tag }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - name: api
            image: mytool-api
            context: api
            dockerfile: api/Dockerfile
          - name: web
            image: mytool-web
            context: web
            dockerfile: web/Dockerfile
    steps:
      - uses: actions/checkout@v4

      - name: Compute image tag
        id: tag
        run: echo "tag=sha-${GITHUB_SHA}" >> "$GITHUB_OUTPUT"

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          context: ${{ matrix.context }}
          file: ${{ matrix.dockerfile }}
          push: true
          tags: |
            ${{ env.DOCKERHUB_ORG }}/${{ matrix.image }}:${{ steps.tag.outputs.tag }}
            ${{ env.DOCKERHUB_ORG }}/${{ matrix.image }}:latest
          cache-from: type=gha,scope=${{ matrix.image }}
          cache-to: type=gha,scope=${{ matrix.image }},mode=max

  notify:
    needs: build
    if: success()
    runs-on: ubuntu-latest
    steps:
      - name: Notify fybud Deploy
        env:
          DEPLOY_WEBHOOK_URL: ${{ secrets.DEPLOY_WEBHOOK_URL }}
          DEPLOY_WEBHOOK_SECRET: ${{ secrets.DEPLOY_WEBHOOK_SECRET }}
          COMMIT_SHA: ${{ github.sha }}
          COMMIT_MSG: ${{ github.event.head_commit.message || github.event_name }}
          COMMIT_AUTHOR: ${{ github.event.head_commit.author.name || github.actor }}
          COMMITTED_AT: ${{ github.event.head_commit.timestamp || github.event.repository.updated_at }}
          REPO: ${{ github.repository }}
        run: |
          : "${DEPLOY_WEBHOOK_URL:?DEPLOY_WEBHOOK_URL secret is required}"
          : "${DEPLOY_WEBHOOK_SECRET:?DEPLOY_WEBHOOK_SECRET secret is required}"
          TAG="sha-${COMMIT_SHA}"
          PUSHED_AT="${COMMITTED_AT:-$(date -u +%Y-%m-%dT%H:%M:%SZ)}"
          payload=$(jq -n \
            --arg tool "$TOOL" \
            --arg tag "$TAG" \
            --arg repo "$REPO" \
            --arg commit "$COMMIT_SHA" \
            --arg commitMessage "$COMMIT_MSG" \
            --arg commitAuthor "$COMMIT_AUTHOR" \
            --arg pushedAt "$PUSHED_AT" \
            --arg api "${DOCKERHUB_ORG}/mytool-api:${TAG}" \
            --arg web "${DOCKERHUB_ORG}/mytool-web:${TAG}" \
            '{tool:$tool,tag:$tag,repo:$repo,commit:$commit,commitMessage:$commitMessage,commitAuthor:$commitAuthor,pushedAt:$pushedAt,images:{api:$api,web:$web}}')
          echo "$payload"
          curl --fail --silent --show-error --retry 3 --retry-all-errors --retry-delay 2 \
            --connect-timeout 10 --max-time 60 -X POST "$DEPLOY_WEBHOOK_URL" \
            -H "Content-Type: application/json" \
            -H "X-Deploy-Secret: $DEPLOY_WEBHOOK_SECRET" \
            -d "$payload"
```

### Org secrets (GitHub → org → Secrets)

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` | Docker Hub org `fybud` |
| `DEPLOY_WEBHOOK_URL` | `https://api.deploy.fybud.com/webhooks/github-actions` |
| `DEPLOY_WEBHOOK_SECRET` | Same secret as Deploy VPS `.env` |

Adjust the matrix `context` / `dockerfile` / image names to match your folders. Keep
`concurrency` so two pushes never race the VPS.

---

## 3. First deploy in the UI

1. Register the tool slug in Deploy (`tools.ts` or **Settings → Tool specs**).
2. Push `main` (Actions builds + webhook).
3. In Deploy UI: paste required env (JWT, OAuth, `CORS_ORIGIN`, and every `*DATABASE*_URL`) —
   not host ports / `IMAGE_TAG` / `VITE_API_BASE_URL` / `PUBLIC_API_URL`.
4. Approve once. Later pushes redeploy automatically.
5. Confirm `https://{tool}.fybud.com`, `https://api.{tool}.fybud.com`, and
   `https://{tool}.fybud.com/env.js` (SPA) shows the API base URL.

### Quick triage (browser “Failed to fetch”)

| Symptom | Likely cause |
|---|---|
| Console `ERR_NAME_NOT_RESOLVED` on `api.*` or `/env.js` | Local DNS / VPN — API and env.js must resolve |
| Login posts to `{tool}.fybud.com/api/...` | `env.js` missing → SPA fell back to same-origin |
| API `401` / `403` after DB restore | Stale JWT or user missing/`isActive=false` in that DB — sign out, re-login, check `User` rows |
| Origin HTTPS fails only for one renamed hostname | LE cert SAN incomplete — rebuild deploy-api or expand cert SANs on the VPS |

## 4. Operating a deployment

Before approving a first deploy, confirm the compose validator passes, every
required environment variable is present, and the images are immutable `sha-*`
tags from the successful Actions build. During a deploy, the UI should show the
current stage and stream its logs. Do not call it live merely because Docker
started a container: success requires the host probe and public HTTPS check.

If a run fails, use the stage shown in the run record:

| Failed stage | Safe response |
|---|---|
| Compose / policy | Fix the repository compose and push a new commit. No production change should have occurred. |
| Image pull | Retry only after the image is present in Docker Hub; keep the current release. |
| Health / smoke | Inspect candidate logs; automatic rollback should restore the prior image **and its environment snapshot**. |
| DNS / TLS | Existing correct DNS may continue serving; a new domain remains pending until DNS and TLS are verified. |
| Nginx | Restore the previous site config and reload only after `nginx -t` passes. |

Manual rollback selects a previous successful release. It must restore its image
digest, compose revision, ports, and encrypted environment snapshot together;
rolling back only `IMAGE_TAG` can leave an incompatible configuration or secret.

## 5. UI requirements

The project page should make risk and recovery obvious:

- One timeline with queued, validation, pull, candidate, health, DNS, TLS,
  nginx, public-check, rollback, and final states; show timestamps and duration.
- A prominent **current release** card (digest, commit, compose revision, secret
  version, deployment time) next to the candidate; never label a project
  “healthy” from Docker state alone.
- A failure card with the failed stage, actionable error, preserved previous
  release, retry eligibility, and a one-click safe rollback/retry where valid.
- Domains should separately show desired vs observed DNS, origin HTTPS,
  Cloudflare/public HTTPS, certificate expiry, and last successful check.
- Surface host capacity (disk, RAM, CPU), queued jobs, image retention, and
  drift warnings before the user presses Deploy.

---

## 6. Checklist

- [ ] `DEPLOY.md` at repo root (this file)
- [ ] `docker-compose.deploy.yml` at repo root with `fybud.expose` / `fybud.domain` / `fybud.health`
- [ ] Every pasteable / injected variable declared under `environment:`
- [ ] Public URL keys (`VITE_API_BASE_URL`, …) declared empty — not in `requiredEnv`
- [ ] SPA ships runtime `/env.js` from container `VITE_*` (not bake-time Hub URLs)
- [ ] Public ports are `127.0.0.1:${*_HOST_PORT}:…`
- [ ] Private services have no ports and no expose labels
- [ ] `.github/workflows/build-push.yml` with Hub push + Deploy webhook + `concurrency`
- [ ] Tool registered in Deploy; secrets pasted; first approve done
- [ ] Public HTTPS and the relevant application smoke check passed
- [ ] Previous release is retained and available for rollback

Verify from the Deploy repo: `node scripts/verify-all.mjs` and `node scripts/smoke-robust.mjs`.

Org seed (maintainers): from the Deploy repo, with a GitHub token that can write
`Fybud/org-defaults` and `Fybud/fybud-app-template`:

```bash
GITHUB_TOKEN=ghp_… node scripts/seed-org-deploy-md.mjs
```

That upserts root `DEPLOY.md` + `AGENTS.md` only (no compose / no `infra/`).
