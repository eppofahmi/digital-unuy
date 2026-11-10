# Environment Variables: Digital UNUY — Website & Digital Service Portal

**Status:** Draft · **Version:** 0.2 · **Source:** `docs/IMPLEMENTATION_PLAN.md` §8 / `docs/PRD.md` §12 · **Service split:** Phase 1 as-built = monolith (Nuxt Nitro + MongoDB + local upload volume); target = `fe/` (Nuxt) + `be/` (Go) per D-STACK

## 0. Target split (D-STACK) — draft

Phase 1 reads everything through Nuxt `runtimeConfig` with a `NUXT_` prefix. Once the Go service exists, that prefix no longer applies to the backend and the variables must be split. **Names below are the proposed mapping, not an approved contract**; final names are fixed during migration wave 0 (plan §13) and then this section replaces §1–§3.

**Backend `be/` (Go):**

| Var | Purpose | Notes |
|---|---|---|
| `PORT` | HTTP listen port | Container/runtime |
| `APP_ENV` | `development` \| `staging` \| `production` | Replaces `NODE_ENV` semantics |
| `MONGODB_URI` | MongoDB connection string | was `NUXT_MONGODB_URI` |
| `MONGODB_DB_NAME` | Database name | new (explicit, per house standard) |
| `SESSION_SECRET` | Session/token key | Session tokens stay random + hashed |
| `SESSION_TTL_HOURS` | Session TTL | Phase 1 is 7 days |
| `INTERNAL_TOKEN` | Header expected from the FE BFF | Only if BFF auth is chosen (plan §13 M1) |
| `CORS_ALLOWED_ORIGINS` | Allowed FE origin(s) | Only if direct cross-origin calls are chosen |
| `PUBLIC_SITE_URL` | Canonical URL for links/email | was `NUXT_PUBLIC_SITE_URL` (server use) |
| `UPLOAD_DIR` **or** `STORAGE_DRIVER`/`GCS_BUCKET` | Upload destination | Depends on plan §13 M2 |
| `SMTP_HOST`, `SMTP_PORT`, `SMTP_SECURE`, `SMTP_USER`, `SMTP_PASS`, `EMAIL_FROM` | Email | were `NUXT_SMTP_*`, `NUXT_EMAIL_FROM` |
| `SEED_ADMIN_PASSWORD`, `SEED_PETUGAS_PASSWORD` | Dev seed only | Never set in production |

**Frontend `fe/` (Nuxt):**

| Var | Purpose | Notes |
|---|---|---|
| `NUXT_PUBLIC_SITE_URL` | Canonical URL, sitemap, email links | unchanged |
| `NUXT_PUBLIC_NOINDEX` | Force noindex on staging | unchanged |
| `NUXT_API_BASE_URL` | Internal BE base URL used by BFF routes | **not** `NUXT_PUBLIC_*` — must not reach the browser |
| `NUXT_INTERNAL_TOKEN` | Token sent to the BE BFF routes | server-only runtimeConfig |
| `NUXT_SESSION_COOKIE_NAME` | Session cookie name | optional, if configurable |

**Rules carried over unchanged:** `.env` is gitignored; secrets never use `NUXT_PUBLIC_`; fail-fast on missing required vars; seed passwords are development-only; the upload volume is backed up with MongoDB.

## 1. App Service (Nuxt Nitro) — Phase 1 as-built

All application config is read through Nuxt `runtimeConfig` or `process.env`. `NUXT_PUBLIC_*` values are exposed to the browser.

| Var | Purpose | Dev | Staging | Prod | Secret | Required at startup |
|-----|---------|-----|---------|------|--------|---------------------|
| `NUXT_MONGODB_URI` | MongoDB connection string | `mongodb://127.0.0.1:27017/digital_unuy` | staging URI | prod URI | Yes | Yes |
| `NUXT_PUBLIC_SITE_URL` | Canonical URL, sitemap, email links | `http://localhost:3000` | staging URL | `https://<domain-produksi>` | No | Yes |
| `NUXT_SESSION_SECRET` | Deployment secret; sessions use random hashed tokens, so this is not used for signing | dev value | staging secret | prod secret | Yes | Yes |
| `NUXT_UPLOAD_DIR` | Private upload directory (relative or absolute) | `.data/uploads` | persistent volume | persistent volume | No | Yes |
| `NUXT_PUBLIC_NOINDEX` | Force noindex on staging even with `NODE_ENV=production` | unset | `true` | unset | No | No |
| `NUXT_SMTP_HOST` | SMTP host; empty → dev outbox / prod email failure | empty | staging SMTP | campus SMTP | No | Prod: yes for email |
| `NUXT_SMTP_PORT` | SMTP port | `587` | `587` | campus port | No | No |
| `NUXT_SMTP_SECURE` | `true` for implicit TLS | `false` | per provider | per provider | No | No |
| `NUXT_SMTP_USER` | SMTP username | empty | staging user | prod user | No | No |
| `NUXT_SMTP_PASS` | SMTP password | empty | staging password | prod password | Yes | No |
| `NUXT_EMAIL_FROM` | From header | local sender | staging sender | official campus sender | No | No |
| `SEED_ADMIN_PASSWORD` | Dev seed override | optional | — | never set | Yes | No |
| `SEED_PETUGAS_PASSWORD` | Dev seed override | optional | — | never set | Yes | No |

## 2. Public / Client-Exposed

| Var | Purpose | Dev | Staging | Prod | Secret |
|-----|---------|-----|---------|------|--------|
| `NUXT_PUBLIC_SITE_URL` | Browser canonical and links | `http://localhost:3000` | staging URL | production URL | No |
| `NUXT_PUBLIC_NOINDEX` | Staging noindex switch (read server-side, but public-safe) | unset | `true` | unset | No |

## 3. Runtime / Deployment

| Var | Purpose | Dev | Staging | Prod | Secret |
|-----|---------|-----|---------|------|--------|
| `NODE_ENV` | Runtime mode; controls indexability and secure cookies | `development` | `production` | `production` | No |
| `PORT` / `NITRO_PORT` | Listen port | `3000` | `3000` | `3000` or proxy target | No |
| `NITRO_HOST` | Bind address | `localhost` | `0.0.0.0` | `0.0.0.0` | No |

## 4. Secrets Handling

- **Local:** `.env` from `.env.example`; `.env` is gitignored and never committed.
- **Staging/production:** environment variables injected by the process (systemd `EnvironmentFile`, Docker Compose `env_file`, or the platform secret store). `.env` permissions `0600`.
- **Never expose:** `NUXT_MONGODB_URI`, `NUXT_SESSION_SECRET`, `NUXT_SMTP_PASS`, `SEED_*_PASSWORD`. None may use `NUXT_PUBLIC_`.
- **Seed passwords:** development only. Never set `SEED_*` in production, and never run `seed:reset` there.
- **Upload volume:** must be writable by the app user and backed up together with MongoDB.

## 5. `.env.example` (authoritative template)

```dotenv
# Application
NUXT_MONGODB_URI=mongodb://127.0.0.1:27017/digital_unuy
NUXT_PUBLIC_SITE_URL=http://localhost:3000
NUXT_SESSION_SECRET=development-only-replace-with-long-random-secret

# Private upload storage (relative to project root)
NUXT_UPLOAD_DIR=.data/uploads

# SMTP (optional in development; required for real email delivery)
# NUXT_SMTP_HOST=smtp.example.com
# NUXT_SMTP_PORT=587
# NUXT_SMTP_SECURE=false
# NUXT_SMTP_USER=
# NUXT_SMTP_PASS=
# NUXT_EMAIL_FROM="Direktorat Transformasi Digital UNU Yogyakarta <no-reply@example.ac.id>"

# SEO — set true on staging to force noindex even with NODE_ENV=production
# NUXT_PUBLIC_NOINDEX=true

# Seed passwords (development only)
# SEED_ADMIN_PASSWORD=Admin#Unu2026
# SEED_PETUGAS_PASSWORD=Petugas#2026
```

## 6. Validation Gaps

| Gap | Impact | Planned fix |
|-----|--------|-------------|
| `NUXT_MONGODB_URI` not validated at boot | First request fails with a connection error | Fail fast at Nitro startup |
| `NUXT_PUBLIC_SITE_URL` not validated | Broken canonical, sitemap, and email links | Parse as https URL in production |
| `NUXT_SESSION_SECRET` not used for signing | Misleading name; rotation has no session effect | Document or remove; sessions already hashed |
| `NUXT_UPLOAD_DIR` not checked for writability | Uploads fail at runtime | Startup write test |
| SMTP vars partly validated | Production email silently fails | Fail fast for activation email path or expose integration status |
| `NUXT_PUBLIC_NOINDEX` absent from `.env.example` upstream | Staging may be indexed if operator forgets | Added to the template above |
| No secret manager | Secrets in `.env` | Docker secrets / campus vault in Phase 3 |

## 7. Open Questions

- **Q1:** final staging/production domains and whether admin shares the same origin.
- **Q2:** SMTP provider, port, TLS mode, and official sender address.
- **Q3:** upload volume path, size, backup, and restore ownership.
- **Q4:** whether `NUXT_SESSION_SECRET` should be removed or repurposed.
- **Q5:** secret store choice for Phase 3.
