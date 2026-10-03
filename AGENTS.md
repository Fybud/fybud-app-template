# Fybud — AI / agent deploy rules

**Copy this file to the root of every new Fybud repo** (or keep it as the default agent context).
Follow it for all infrastructure and deploy work. Humans follow `DEPLOY.md` (same folder) for the
step-by-step Actions + compose templates.

## Goal

Push to `main` on a Fybud GitHub org repo → GitHub Actions builds/pushes Docker images →
**Fybud Deploy** pulls them, allocates free host ports, writes nginx, creates Cloudflare DNS if
missing → `https://{tool}.fybud.com` is live.

First time only: paste secrets in Deploy UI and approve. Later pushes redeploy automatically. This
is Fybud's **own control plane** — no Coolify, no CapRover, no Traefik.

## Repo layout

```
MyTool/
  DEPLOY.md                 ← human playbook (Actions + compose templates)
  AGENTS.md                 ← this file
  README.md                 ← optional product README
  .github/workflows/build-push.yml
  docker-compose.deploy.yml ← ONLY file Deploy downloads (repo root; no full clone)
  <app code folders>        ← api/, web/, worker/ etc. — NOT separate GitHub repos
```

- One GitHub repo = one product (frontend + backend + workers as **folders**).
- Compose lives at the **repo root** as `docker-compose.deploy.yml` only (not `.yaml`).
- Do **not** create `infra/`, `infra/demo/`, `infra/{company}/`, or hand-written nginx configs.
- Private sidecars (workers, redis, classifiers): own services in the same compose **or** a separate
  compose project with **no** public labels.

## `docker-compose.deploy.yml` contract

### Public service (needs domain)

```yaml
services:
  web:
    image: fybud/mytool-web:${IMAGE_TAG:?IMAGE_TAG required}
    ports:
      - "127.0.0.1:${WEB_HOST_PORT}:5173"   # container port fixed; HOST port filled by Deploy
    labels:
      fybud.expose: "true"
      fybud.domain: mytool.fybud.com
      fybud.role: web
      fybud.health: "any"                    # SPA / static — process-up only
    networks: [internal]

  api:
    image: fybud/mytool-api:${IMAGE_TAG:?IMAGE_TAG required}
    ports:
      - "127.0.0.1:${API_HOST_PORT}:4100"
    labels:
      fybud.expose: "true"
      fybud.domain: api.mytool.fybud.com
      fybud.role: api
      fybud.health: "/health"                 # must match the compose healthcheck path
    healthcheck:
      test: ["CMD-SHELL", "node -e \"fetch('http://127.0.0.1:4100/health').then(r=>process.exit(r.status<400?0:1)).catch(()=>process.exit(1))\""]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 20s
    networks: [fybud-net]
```

### Private service (no internet)

```yaml
  worker:
    image: fybud/mytool-api:${IMAGE_TAG:?IMAGE_TAG required}
    command: ["worker"]
    # NO ports:
    # NO fybud.expose / fybud.domain
    networks: [fybud-net]
```

### Labels (only these)

| Label | When | Meaning |
|---|---|---|
| `fybud.expose: "true"` | Public only | Publish + DNS + nginx |
| `fybud.domain` | Required if expose | FQDN e.g. `cep.fybud.com` |
| `fybud.health` | **Required** if expose | How Deploy probes the host port after `compose up` |
| `fybud.role` | Optional | `web` / `api` — disambiguates which domain is which for env computation |

`fybud.health` values:

| Value | Behaviour |
|---|---|
| `"/health"` (or `"/api/health"`) | **Strict**: host probe requires HTTP < 400 (4xx/5xx → fail → auto-rollback) — public APIs |
| `"any"` | Process-up only: any HTTP response counts — web/SPA/static, or APIs with no health route |
| *(absent)* | Legacy lenient probe of `/` — Deploy logs a warning telling you to declare the label |

Public APIs declare a compose `healthcheck:` **and** a matching strict `fybud.health` path.
Public web uses `"any"`. Deploy gates on containers **running**, then the host HTTP probe —
Docker's own HEALTHY/starting status is not a hard fail.

### Env: declare every variable in the compose file

`env_file: [.env]` tells Docker to load keys, but it does **not** say *which* keys must exist — a
forgotten variable then only shows up as a container crash loop. So every variable a service reads
must **also** be declared under `environment:`, and the value itself says where it comes from:

```yaml
    env_file: [.env]
    environment:
      PORT: 4100                        # literal — fixed in compose (or empty paste)
      JWT_SECRET:                       # empty — PASTE in the Deploy UI (requiredEnv)
      INTENT_CLASSIFIER_URL: http://intent-classifier:8091   # literal — fixed here
      DATABASE_URL:                     # empty — PASTE postgres/postgres@fybud-postgres/<db>
```

| Value in `environment:` | Meaning |
|---|---|
| `PORT: 4100` | **Literal** — fixed in the compose file |
| `JWT_SECRET:` / `DATABASE_URL:` (empty) | **Paste** in the Deploy UI. The empty value *is* the flag; `verify-all.mjs` cross-checks against `requiredEnv` |
| `VITE_API_BASE_URL:` / `PUBLIC_API_URL:` / `PLATFORM_*_BASE_URL:` (empty) | **Deploy-injected** from `fybud.domain` — declare empty, never paste, never list in `requiredEnv` |
| `${VAR:?message}` | **Deploy-injected** (`*_HOST_PORT`, `IMAGE_TAG`) — never pasted |

There is **no optional** form. Do not use `${KEY:-}`, `${KEY:-default}`, or `optionalEnv`.
Every key is either a literal in compose, empty (paste), empty (public-URL inject), or `${:?}` (port/tag inject).

- An empty declaration is not a blank value: Deploy runs `docker compose --project-directory <tenant
  dir>`, so the entry resolves from the runtime `.env` it writes — the pasted value still reaches the
  container, and a key nobody pasted arrives empty instead of clobbering `env_file`.
- `environment:` is the **manifest of what must be pasted**; keep `env_file` for the values. This is
  also the per-service list a human reads to know what the Deploy UI still needs.
- Deploy refuses to deploy without those keys: approve and env-save answer `400` with a `missing[]`
  list, and the runner re-checks before `compose up` — a push-to-`main` redeploy that lost a key fails
  **naming it** instead of booting a container with it unset.
- The empty **paste** declaration and the spec's `requiredEnv` are one list kept in two places, and
  `verify-all.mjs` fails either drift: a tool whose spec `requiredEnv` key the compose never declares,
  **and** a tool whose empty paste declaration the spec's `requiredEnv` does not list.
  Public URL inject keys are excluded from that paste cross-check.
- Vite SPAs must ship runtime `/env.js` from container `VITE_*` (see `DEPLOY.md`) — do not bake
  production API hosts into the image at CI build time.
- `verify-all.mjs` also fails `${KEY:-…}` in `environment:`, a service with `env_file` but no
  `environment:`, an empty `*_HOST_PORT` / `IMAGE_TAG` (injected, never pasted), and an `AGENTS.md`
  that drifted from this canonical file.

**Do not** put host ports in labels. Deploy:

1. Finds free host ports on the VPS
2. Writes `WEB_HOST_PORT=…` / `API_HOST_PORT=…` into runtime `.env`
3. Compose becomes `127.0.0.1:<host>:<container>`
4. Nginx: `domain → http://127.0.0.1:<host>`
5. Cloudflare: create A record if missing (VPS IP from Deploy settings)

### Network / DB

- Shared Postgres container hostname: `fybud-postgres`. Superuser is **`postgres` / `postgres`**.
  You must paste every DB URL yourself (`DATABASE_URL`, `PLATFORM_DATABASE_URL`, etc.) in the exact format:
  `postgresql://postgres:postgres@fybud-postgres:5432/<dbname>`.
  Deploy will **create the database if it is missing** when you save the env, but you are responsible for supplying this exact URL string. Any other user/password will be rejected.
  The DB name need not match the tool slug, and **multiple projects may share one database.** The Deploy UI **Databases** page lists DBs and project mapping.
- Never run a Postgres service inside the tool compose unless explicitly required and private.
- **Network isolation:** only Postgres-facing services join external `fybud-net`. Every other service
  (web frontends, admin UIs, crawlers) stays on a project-local network:
  ```yaml
  networks:
    fybud-net: { name: fybud-net, external: true }
    internal: { driver: bridge }
  ```
- **Healthchecks:** every API service gets a compose `healthcheck:` against its health endpoint
  (`/health`, `/api/health`, …) **and** a matching `fybud.health` label. Deploy's gate requires
  containers **running**, then probes `127.0.0.1:<hostPort><fybud.health>` from the host (authoritative).
  Docker's own HEALTHY/starting status is not a hard fail — SPA/web images often lack wget/curl.
  Use `fybud.health: "any"` when the service has no real health route.
- **State:** any service that writes to disk (uploads, media) must declare a named volume —
  containers are recreated on every deploy and local files are wiped.

## CI

- Workflow builds images → Docker Hub org `fybud` (`fybud/<service>:sha-…` + `:latest`).
- Notify Deploy webhook with tool slug + tag.
- Cancel superseded builds so two pushes never race the VPS:
  ```yaml
  concurrency:
    group: build-push-${{ github.ref }}
    cancel-in-progress: true
  ```
- Org secrets: `DOCKERHUB_*`, `DEPLOY_WEBHOOK_URL=https://api.deploy.fybud.com/webhooks/github-actions`, `DEPLOY_WEBHOOK_SECRET` (public repos on free GitHub org plan).
- Full workflow template: see `DEPLOY.md`.

## Deploy UI

- Paste app secrets once (JWT, OAuth, etc.) — the onboarding checklist on the project page shows
  exactly which required variables are still missing.
- Paste keys per tool live in **Deploy → Settings → Tool specs** → `requiredEnv` (stored in the Deploy
  database). Approving or saving env is rejected with a `missing[]` list when a paste var is absent.
- Live deploy output streams into the project's Logs tab
  (`/api/projects/:tool/runs/:id/stream`, Server-Sent Events).
- Do **not** paste host ports, `IMAGE_TAG`, or public URL injects (`VITE_API_BASE_URL`,
  `PUBLIC_API_URL`, `PLATFORM_*_BASE_URL`) — Deploy writes those from `fybud.domain`. **Do** paste
  every `*DATABASE*_URL` as `postgres`/`postgres`@`fybud-postgres`, plus app secrets / `CORS_ORIGIN`.
- Control-plane secrets (Cloudflare, Hub, webhook) live in Deploy’s own `.env` on the VPS, not in tool repos.
- After renaming domains, rebuild deploy-api if ssl: still skips certbot without SAN expand —
  tool redeploys do not update the control plane.

## Domain convention

- Web: `{tool}.fybud.com`
- API: `api.{tool}.fybud.com`
- No company/tenant subfolders for now.

## What agents must not do

- Do not invent Coolify / CapRover / Traefik as the edge unless explicitly asked — Fybud uses its **own control plane (host nginx + dynamic host ports + Cloudflare DNS + certbot)**.
- Do not add a `captain-definition*`, a PaaS manifest, or hand-written nginx — there is no third-party PaaS in this stack.
- Do not hardcode host port numbers in compose (no `9000:5173` without `${…_HOST_PORT}`).
- Do not add an `infra/` folder for deploy compose, or per-tenant `infra/{customer}` trees.
- Do not commit `.env` secrets.
- Do not expose workers/redis/DB on host ports.

## Checklist for a new tool

1. Add root `DEPLOY.md` (human playbook) and `AGENTS.md` (this file).
2. Add root `docker-compose.deploy.yml` with expose/domain/health labels + `${*_HOST_PORT}` port lines, and declare every variable under `environment:`.
3. Add Actions build-push → `fybud/*` images + Deploy webhook (`concurrency` set) — template in `DEPLOY.md`.
4. Register tool slug in Fybud Deploy `tools.ts` (or edit its spec in Settings → Tool specs).
5. Push `main` → approve once in Deploy UI with env → later pushes auto-deploy.
6. Verify from the Deploy repo: `node scripts/verify-all.mjs` (contract) and
   `node scripts/smoke-robust.mjs` (end-to-end).
