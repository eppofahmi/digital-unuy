# Deployment: Digital UNUY — Website & Digital Service Portal

**Status:** Draft (future plan — no artifacts committed) · **Version:** 0.1 · **Platform:** VM + Docker Compose (Implementation Plan §3) · **Owner:** Arief Faqihudin

## 1. Scope

This runbook covers shipping the Phase 1 Nuxt/Nitro application, MongoDB, and the private upload directory to a campus-managed VM with Docker Compose, plus staging/production separation, seed/migration order, email, backups, rollout, and rollback.

**Out of scope:** SSO, object storage, malware scanning (Phase 2), Redis/multi-instance (Phase 3), and platform selection (locked in plan §3).

**Current state:** no deploy artifacts are committed. This document is the **target runbook for future production**; Dockerfile, compose, and GitLab CI will be added when the project moves to GitLab UNU and implementation starts. Nothing here is active today.

**Source documents:** `docs/IMPLEMENTATION_PLAN.md` §3/§8/§11 · `docs/ENVIRONMENT_VARIABLES.md` · `docs/API_CONTRACTS.md` · `docs/TEST_PLAN.md` §7/§9 · `docs/SEO.md`.

## 2. Deployable Inventory

| Service | Type | Runtime | Build Command | Port | Healthcheck |
|---------|------|---------|---------------|------|-------------|
| `app` | Nuxt Nitro (SSR + API) | Node 20+ | `npm ci` → `npm run build` → `node .output/server/index.mjs` | 3000 | `GET /` (temporary; dedicated `/health` planned) |
| `mongodb` | Database | MongoDB 7+ | Managed or container | 27017 (internal) | `mongosh --eval "db.adminCommand('ping')"` |
| Upload volume | Private files | Filesystem | Persistent volume at `NUXT_UPLOAD_DIR` | — | Writable check |
| Reverse proxy | TLS + host routing | Campus Nginx/Caddy or Cloudflare | Not managed here | 80/443 | Proxy-level |

One application deployable. Uploads and MongoDB must both persist.

**Target split (D-STACK):** after the migration in `IMPLEMENTATION_PLAN.md` §13, the stack becomes two deployables. This table is the intended inventory, not a committed artifact.

| Service | Type | Exposure | Notes |
|---|---|---|---|
| `fe` | Nuxt 3 (SSR + BFF API routes) | Public ingress, TLS at proxy | Owns the session cookie and CSRF/internal-token checks |
| `be` | Go + Gin (JSON API) | **No public ingress** | Reached only by `fe` (internal network on VM, or authenticated/internal Cloud Run) |
| `mongodb` | MongoDB 7+ | Internal only | Single database shared by both during and after the split |
| Upload volume | Private files | Mounted to the upload owner | Either stays with `fe` behind BFF routes, or moves to object storage (plan §13 M2) |

Platform choice (VM + Compose vs Cloud Run) is still Q2 and does not change the exposure rule: only the Nuxt FE is publicly reachable. The `be` service must never accept browser traffic directly, because the session cookie and CSRF origin check live on the FE.

## 3. Containerization (planned)

| Concern | Implementation |
|---------|----------------|
| Base image | `node:20-alpine` or `node:22-alpine` |
| Build | `npm ci`, `npm run build`; output `.output/` |
| Runtime | `node .output/server/index.mjs`, non-root user |
| Port | `PORT=3000`, `NITRO_HOST=0.0.0.0` |
| Uploads | Named volume mounted at `NUXT_UPLOAD_DIR`; **not** inside the image |
| Secrets | `.env` / platform secret store; never baked into layers |
| Healthcheck | `GET /` or `/api/site`; a dedicated `/health` is a Phase 3 task |
| Start period | Allow ≥20s for Nuxt/Nitro and MongoDB connection |

**Build locally (future):**

```bash
npm ci
npm run build
NODE_ENV=production node .output/server/index.mjs
```

## 4. Topology (planned)

```
internet → campus reverse proxy / Cloudflare (TLS, domain)
              → app:3000 (Nuxt Nitro SSR + API)
                  → mongodb:27017 (internal)
                  → upload volume (private files)
```

| Service | Restart | Network | Volume | Notes |
|---------|---------|---------|--------|-------|
| `app` | `unless-stopped` | `internal` | `uploads:/data/uploads` | Bind `127.0.0.1:3000`; only proxy reaches it |
| `mongodb` | `unless-stopped` | `internal` | `mongo:/data/db` | Not published; auth enabled |
| Reverse proxy | — | host | — | TLS, `Host`/`X-Forwarded-*`, body size ≥12 MB |

**Single instance only:** rate limits are in-memory and uploads are local. Do not run replicas until object storage + Redis land (Phase 2/3).

## 5. Config & Secrets

Every variable comes from `docs/ENVIRONMENT_VARIABLES.md`. Staging and production have separate MongoDB databases, upload directories, SMTP settings, and secrets.

| Var | Staging | Production | Secret | Provisioning |
|-----|---------|------------|--------|--------------|
| `NUXT_MONGODB_URI` | staging DB | prod DB | Yes | `.env` (0600) / secret store |
| `NUXT_PUBLIC_SITE_URL` | staging URL | production URL | No | `.env` |
| `NUXT_SESSION_SECRET` | random | random | Yes | `.env` / secret store |
| `NUXT_UPLOAD_DIR` | `/data/uploads` | `/data/uploads` | No | Volume mount |
| `NUXT_PUBLIC_NOINDEX` | `true` | unset | No | `.env` |
| `NUXT_SMTP_*` | staging SMTP | campus SMTP | Pass yes | `.env` / secret store |
| `NUXT_EMAIL_FROM` | staging sender | official sender | No | `.env` |
| `SEED_*_PASSWORD` | optional | **never** | Yes | — |

**Rules:** `.env` gitignored, `chmod 600`; no secret in the image; `NODE_ENV=production` enables secure cookies and indexability policy.

## 6. Seed & Data Migrations

Scripts are application-level, not a migration framework.

| Command | Behavior | Production |
|---------|----------|------------|
| `npm run seed` | Idempotent seed for main entities | Do not run blindly; verify |
| `npm run seed:reset` | **Drops the application database and reseeds** | **Never run** |
| `npm run seed:team-members` | Seed team data | Review first |
| `npm run migrate:ecosystem` | Ecosystem data migration | Review first |
| `npm run migrate:team-groups` | Team group restructure | Review first |

**Order for a new environment:**

1. Back up MongoDB and the upload directory (if existing).
2. Create the database/user and set `NUXT_MONGODB_URI`.
3. Run the required seed/migration scripts once, in staging first.
4. Verify via `/api/site`, `/api/home`, `/api/services`.
5. Start the app.

**Rollback:** restore the MongoDB dump and upload directory together. There are no down migrations; scripts must be idempotent or accompanied by a backup.

## 7. CI/CD (planned)

| Stage | Command | Trigger | Gate |
|-------|---------|---------|------|
| `validate` | `npm ci` → `npm run build` → E2E/visual/responsive (server running) | MR and default branch | Block merge |
| `build_image` | Docker build → push `:$SHA` and `:latest` | Default branch | — |
| `deploy_staging` | SSH → pull → seed/migrate if needed → `up -d` | Default branch | Automatic |
| `deploy_production` | Same | Tag/manual | Approval by release owner |

**Required CI variables:** `CI_REGISTRY_*`, `SSH_PRIVATE_KEY`, `DEPLOY_HOST`, `DEPLOY_USER`, `DEPLOY_PATH`, `STAGING_URL`, `PRODUCTION_URL`.

**Gaps before trusting the pipeline:** GitLab UNU migration is future; no dependency/secret scanning yet; E2E scripts require a running server and MongoDB; no image cache.

## 8. Domains & TLS

| Service | Domain | Target | TLS |
|---------|--------|--------|-----|
| Public site + `/admin` | institutional domain (e.g. `digital.unu-jogja.ac.id`) | Reverse proxy → `127.0.0.1:3000` | Campus certificate or Cloudflare **Full** |

**Rules:**

- Public site and admin share one origin; session cookie is `SameSite=Lax` and CSRF compares Origin to host.
- Proxy must forward `Host`, `X-Forwarded-Host`, `X-Forwarded-For`, `X-Forwarded-Proto` and **overwrite** incoming `X-Forwarded-For` (rate limits and audit IP).
- Allow request bodies up to at least **12 MB** for request attachments and 10 MB document uploads.
- Do not cache `/admin`, `/lacak`, `/api`, or authenticated responses.
- CORS: none; same-origin only.

## 9. Observability & Health

| Concern | Phase 1 | Phase 3 target |
|---------|---------|----------------|
| Health | `GET /` / `/api/site` | Dedicated `/health` with Mongo check |
| Logs | `docker compose logs` / journald | Centralized logs |
| Email | `NotificationLog` in MongoDB | Alert on failed rate |
| Uptime | External probe | Uptime monitor |
| Errors | Console logs | Error tracking |
| Disk | Manual | Alert on upload volume and Mongo disk |
| SEO | `robots.txt`, sitemap, Search Console | Core Web Vitals monitoring |

**Minimum pre-launch alarms:** app restarts >3/10 min; Mongo unavailable; upload volume >80%; failed notification rate; public probe 5xx; certificate expiry.

## 10. Rollout & Rollback

**Rollout order:** CI validate → staging deploy → staging smoke (login, submit, track, admin lifecycle, document download) → production approval → production deploy → production smoke.

**Deploy (planned):**

```bash
docker compose pull app
docker compose up -d app
docker compose ps
```

**Rollback (app):** redeploy the previous immutable image tag and keep the same Mongo/upload volumes.

```bash
APP_IMAGE=<previous-sha> docker compose up -d app
```

**Rollback (data):** stop the app; restore the MongoDB dump and upload directory snapshot together; start the app; verify `/api/home` and one document download.

**Post-deploy smoke:**

1. `GET /` → 200 and SSR content.
2. `GET /api/site` → settings.
3. `GET /robots.txt` → correct environment policy.
4. Submit a staging test request → ticket + tracking link.
5. Track it → detail renders.
6. Admin login → dashboard; verify/start/complete the test request.
7. Download a published document.
8. Confirm a private route returns `noindex` and admin requires auth.

**Abort criteria:** any 5xx on public pages, upload failure, tracking mismatch, login failure, or missing `noindex` on private routes.

## 11. Document Info & Sign-off

| Field | Value |
|-------|-------|
| Status | Draft (future plan — no artifacts committed) |
| Version | 0.1 |
| Platform | VM + Docker Compose (plan §3) |
| Source plan / contracts | Implementation Plan v0.1 · API/Data/Env contracts v0.1 |
| Author / Owner | Arief Faqihudin |
| Approver + date | Belum ditandatangani |
| File location | `docs/DEPLOYMENT.md` |

**Open questions**

- Q1: Who owns the VM, DNS, TLS, and MongoDB (Direktorat vs unit)?
- Q2: Exact staging/production hostnames and `DEPLOY_PATH`.
- Q3: Cloudflare vs campus reverse proxy; body-size limits.
- Q4: MongoDB managed service vs containerized; backup tooling.
- Q5: Upload volume size, backup destination, and restore-test schedule.
- Q6: SMTP provider and sender identity.
- Q7: When GitLab migration, object storage, Redis, and `/health` land.
