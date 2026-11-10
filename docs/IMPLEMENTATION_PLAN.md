# Implementation Plan: Digital UNUY — Website & Digital Service Portal

**Status:** Draft · **Version:** 0.1 · **Source PRD:** PRD v1.0, `docs/PRD.md` · **Source Brief:** Design Brief v0.1, `docs/DESIGN_BRIEF.md`

## 1. Architecture Decision

| Decision | Choice | Rationale | Status |
|----------|--------|-----------|--------|
| Codebase topology **(target)** | **Split — two repos: `fe/` Nuxt + `be/` Go** | House standard: every project uses Go for the backend and Vue/Nuxt for the frontend. It also moves the API onto the stack SOP §6.3 requires and isolates uploads/email from public SSR. | **Decided 2026-11-07** (D-STACK). Phase 1 code is still the Nuxt monolith below. |
| Phase 1 as-built | Monolith — Nuxt 3 full-stack (Vue SSR + Nitro API) | Shipped and QA'd as one deploy unit. `server/api`, `server/models`, and `server/utils` already isolate domain code from HTTP, which is what makes the port tractable. | Current state; porting surface for the split |
| Service boundary **(target)** | `fe/` — pages, components, layouts, composables, BFF routes · `be/` — handler → service → repository → MongoDB · `API_CONTRACTS.md` as the single contract | The FE keeps SSR/SEO and becomes the only public ingress; the BE serves JSON and is not exposed directly (same BFF pattern as `website-builder` and `survey`). | Redirect target layout in §2, migration in §13 |
| Rendering | SSR for public and admin routes in the FE; BE is JSON-only | SEO policy and auth-gated admin require server rendering; the BE stays stateless HTTP. | Unchanged |

**Decision record:** Phase 1 shipped as a Nuxt monolith because it had one client, one deploy unit, and one MongoDB connection. That is no longer the constraint: the house standard is Go BE + Nuxt FE for all projects, so the split is a standing decision rather than a trigger-based option. Phase 1 remains the reference implementation and stays deployable until the migration in §13 lands; this document therefore distinguishes **as-built** (Nitro monolith) from **target** (Go + Nuxt) instead of rewriting history.

**Why the split is not free (must be designed, not assumed):**

- The session cookie (`SameSite=Lax`) and CSRF origin check currently rely on public pages and API sharing one origin. A separate BE needs one of: FE-as-BFF with an internal token (the pattern used by `website-builder`/`survey`), or cross-site cookies with CORS, or bearer tokens.
- Uploads live on the local filesystem (`NUXT_UPLOAD_DIR`). With a stateless Go service, uploads either stay behind the FE (BFF) or move to object storage (already planned in T21).
- Rate limiting is in-process today; two services make it two independent limiters unless a shared store (Redis, T22) is introduced.
- `shared/` (types, permissions, SEO) disappears as a shared module; `API_CONTRACTS.md` plus generated Swagger become the contract, and FE types are derived from it.

## 2. Repo & Workspace Layout

```
digital-unuy/
├── pages/                    # public pages + admin pages
│   ├── index.vue, tentang.vue, tentang-kami/tim.vue
│   ├── layanan/, ekosistem-digital/, dokumen/, publikasi/
│   ├── masuk.vue, aktivasi.vue, lacak.vue, pencarian.vue, halaman/
│   └── admin/                # dashboard, permohonan, layanan, dokumen, ekosistem,
│                             # publikasi, halaman, tim, pengguna, seo, notifikasi, audit
├── layouts/                  # default (public), auth, admin
├── components/               # home/*, cards, form builder, editor, SEO, dialogs
├── composables/              # useAuth, useApi, useSite, usePageSeo, useToast, useFormat
├── middleware/               # auth, admin, guest, admin.global
├── server/
│   ├── api/                  # public, track, auth, admin endpoints
│   ├── routes/               # sitemap.xml, robots.txt
│   ├── models/index.ts       # 22 Mongoose models
│   └── utils/                # security, validation, uploads, email, audit, serializers,
│                             # indexing, rate-limit, platform-input
├── shared/                   # types.ts, permissions.ts, seo.ts
├── scripts/                  # seed, migrations, e2e, visual, responsive, asset capture
├── docs/                     # PRD, design brief, plan, contracts, SEO, tests, deploy
├── app.vue, error.vue
├── nuxt.config.ts, tailwind.config.ts
└── package.json
```

**Repo model (as-built):** single repository, one deploy artifact (`.output/`), one MongoDB database. No workspace tooling.

**Target repo model:** two repositories under the product folder — `fe/` (Nuxt) and `be/` (Go) — matching the house structure used by `website-builder` and `survey`. The monolith stays as the reference until the migration is complete.

```
03-website-tim/
├── fe/                          # Nuxt 3 + Vue 3: pages, layouts, components,
│                                # composables, middleware, SSR, and BFF API routes
│   ├── pages/                   # public + admin screens (ported from current pages/)
│   ├── server/api/              # BFF: proxy to be/, session cookie, CSRF/internal token
│   └── nuxt.config.ts
└── be/                          # Go + Gin
    ├── cmd/server/main.go       # entrypoint, dependency wiring
    ├── internal/
    │   ├── config/              # env config, fail-fast validation
    │   ├── delivery/http/       # handler/, middleware/, router.go
    │   ├── domain/              # models + domain errors
    │   ├── repository/          # MongoDB data access
    │   └── service/             # business logic (port of server/utils + models)
    ├── pkg/                     # response, validator, logger, uploads client
    ├── migrations/              # MongoDB index/migration scripts
    ├── docs/                    # generated Swagger (swaggo)
    ├── Makefile
    └── go.mod
```

**Porting surface:** `server/api/**` → `be/internal/delivery/http/handler`; `server/models/**` → `be/internal/domain/models` + `be/internal/repository`; `server/utils/**` → `be/internal/service` + `be/pkg`. The Nuxt `pages/`, `components/`, `layouts/`, and `composables/` move to `fe/` unchanged except for the API base URL and auth transport.

## 3. Deployment Platform & Topology

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Platform | **VM + Docker Compose** (target); Phase 1 runs a bare Node process | SOP requires containers and campus-managed staging/production. Uploads need a persistent volume, and MongoDB can be co-located or managed. No serverless statelessness constraint. |
| Constraint: statelessness | **Not stateless** — local uploads under `NUXT_UPLOAD_DIR` | Container needs a persistent volume. Object storage is a Phase 2 non-goal (PRD D6). |
| Constraint: background jobs | None — email and audit run inline in the request | No queue/worker. Email failure returns 502 on activation; notifications log the failure. |
| Constraint: timeouts | No platform cap (VM) | Reverse proxy timeout must exceed upload/email duration. |
| Constraint: secrets | Environment variables (`.env` on the VM) | No secret manager in Phase 1; `.env` is gitignored. |
| Cost model | Fixed monthly VM | ≤1k staff, ≤5k requests/year. |
| Services | `app` (Node) + MongoDB + persistent upload volume; reverse proxy in front | Single app container. |
| Region | Campus-managed infrastructure, Indonesia | Data residency and institutional ownership per SOP. |
| Domain | Public site on the institutional domain; `/admin` on the same origin | No CORS; session cookie and CSRF Origin check assume same origin. |
| CI/CD | GitLab pipeline: lint/build/test → image → staging → manual prod (future; current remote is GitHub) | SOP requires GitLab, MR review, dependency/secret scanning, release tags. |
| Environments | staging and production with separate MongoDB, upload dirs, and SMTP | SOP §6.3. |

**Topology note:** because uploads are local, Phase 1 must run exactly one app instance. Horizontal scaling requires object storage first, then Redis for rate limits.

## 4. Supporting Architecture

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Backend framework | Nuxt Nitro (h3 handlers, TypeScript) | Co-located with SSR; typed handlers; minimal ops. |
| Frontend framework | Nuxt 3.21 + Vue 3.5 + TypeScript strict | SSR for SEO and admin; SPA navigation. |
| Styling | Tailwind CSS 3.4 + CSS custom properties | Design Brief §3; one token source. |
| Database | MongoDB + Mongoose 9 | Flexible CMS content, form snapshots, request documents. |
| Auth | Email/password, bcrypt cost 12; opaque session token SHA-256 stored; 7-day cookie | Server-side revocation, no JWT secret sprawl. |
| Session secret | `NUXT_SESSION_SECRET` exists but is not used to sign sessions | Documented; random tokens are hashed server-side. |
| Authorization | 11 permission keys; `superadmin` fixed, `admin` editable; 60s role cache | Granular RBAC without a role explosion. |
| File storage | Private local filesystem; random storage keys; MIME + extension + magic-byte validation | No public upload directory. |
| Images | Sharp for team photos (3:4 crop, focal point) | Deterministic public photos. |
| Email | Nodemailer + dev outbox simulation; `NotificationLog` records sent/failed/simulated | Campus SMTP in production. |
| Markdown | `marked` + `sanitize-html` allowlist | CMS content safety. |
| Caching | Role permission cache 60s; site settings cache via `getSettings` | Low invalidation surface. |
| Background jobs | None | Request/response only. |
| Feature flags | None | Fixed Phase 1 scope. |
| Testing | Playwright-core scripts: E2E 105, visual 44, responsive 140; no unit-test runner configured | As-built; unit tests are a Phase 2 gap. |
| API contract | Target `/api/v1` + `{data,message,error}`; Phase 1 `/api/*` raw | PRD D4 / Q-API. |
| API docs | None (OpenAPI planned Phase 3) | Endpoints documented in `API_CONTRACTS.md`. |
| Observability | Console logs; `NotificationLog` and `AuditLog` in DB | Health check, structured logs, uptime monitoring planned Phase 3. |

### 4.1 Target stack (split)

The table above documents Phase 1 as-built. After the split, these are the layers that change; everything else (Nuxt/Vue, Tailwind, MongoDB, Sharp, Markdown pipeline) carries over.

| Decision | Target | Notes |
|---|---|---|
| Backend framework | **Go + Gin** | House standard; replaces Nitro handlers |
| API documentation | **swaggo/swag**, UI at `/swagger`, `make swagger` committed | Mandatory standard; replaces "None (OpenAPI planned Phase 3)" |
| Activity log | Async audit service (goroutine/queue) with the schema in the house standard | Mandatory; complements the existing `AuditLog` collection |
| Error handling | Central `httperr` mapping; domain errors `ErrNotFound`/`ErrForbidden`/`ErrConflict`/`ErrValidation` | No stack traces to clients |
| Data access | Repository pattern over MongoDB (mongo-driver v2) | Mongoose models are ported, not the driver |
| Backend tests | `go test ./...`, `go vet ./...`, `go test -race ./...` | Fills the Phase 1 unit-test gap |
| Frontend | Nuxt 3.21 + Vue 3.5 + TypeScript strict | Unchanged; keeps SSR/SEO |
| Auth transport **(open)** | FE holds the session cookie and calls BE through BFF routes with an internal token; **or** cross-site cookie with CORS | Must be decided before the first endpoint migrates — see §13 |
| Deployment | Two services; FE public, BE internal-only | Platform (VM vs Cloud Run) is Q2 and does not change the topology requirement |

## 5. Task Breakdown

Every task traces to a PRD feature (FR-xx). Status reflects Phase 1 as-built.

| # | Task | Owner | Effort | Priority | Milestone | Status | Acceptance Criteria |
|---|------|-------|--------|----------|-----------|--------|---------------------|
| T01 | Public shell, navigation, footer, design tokens | FE | M | P1 | M1 | Done | 320–1920px; skip link; `Ctrl+K`; token parity (FR-20) |
| T02 | Home + CMS-driven sections | FE+BE | M | P1 | M1 | Done | Hero, services, ecosystem, documents, articles, verified metrics (FR-01) |
| T03 | Service catalog + detail | FE+BE | M | P0 | M1 | Done | Search/category/pagination; detail sections (FR-02) |
| T04 | Request form (dynamic) + multipart submit | FE+BE | L | P0 | M2 | Done | Field types, `showIf`, files, idempotency, ticket (FR-03) |
| T05 | Tracking + requester follow-up | FE+BE | L | P0 | M2 | Done | Ticket+token, timeline, public comments, respond/comments/attachments (FR-04) |
| T06 | Auth, sessions, activation | BE+FE | M | P0 | M2 | Done | bcrypt, rate limit, 48h token, 502 on email failure (FR-05) |
| T07 | Admin dashboard | BE+FE | S | P1 | M3 | Done | Counts, completion rate, average hours, action queue (FR-06) |
| T08 | Request lifecycle + assignment + comments | BE+FE | L | P0 | M3 | Done | State machine, conditional updates, email, audit (FR-07) |
| T09 | Service admin + form builder versioning | BE+FE | L | P0 | M3 | Done | Validation, new version, snapshot, activation guard (FR-08) |
| T10 | Documents + versions | BE+FE | M | P1 | M3 | Done | Upload, private files, publish/archive, versions (FR-09) |
| T11 | Ecosystem CMS | BE+FE | M | P1 | M3 | Done | Categories, platforms, modules, metrics, filters (FR-10) |
| T12 | Publications + pages | BE+FE | M | P1 | M3 | Done | Markdown sanitized, draft/publish, SEO (FR-11) |
| T13 | Team CMS | BE+FE | M | P2 | M3 | Done | Groups, photos, focal point, publish (FR-12) |
| T14 | Users, invitations, roles/permissions | BE+FE | L | P0 | M3 | Done | Nonaktif default, resend, last superadmin guard, role edit (FR-13) |
| T15 | Settings + SEO indexing | BE+FE | M | P1 | M3 | Done | Validation, verified metrics, sitemap/robots policy (FR-14, FR-18) |
| T16 | Audit log | BE+FE | S | P1 | M3 | Done | Metadata sanitization, filters (FR-15) |
| T17 | Notifications | BE+FE | S | P2 | M3 | Done | Action count + list (FR-16) |
| T18 | Global search | FE+BE | M | P1 | M3 | Done | Cross-type search, type filter, counts (FR-17) |
| T19 | Security middleware | BE | M | P0 | M2 | Done | CSRF marker+origin, rate limits, headers, upload validation (FR-19) |
| T20 | QA scripts + responsive/visual checks | Shared | M | P1 | M4 | Done | E2E 105, visual 44, responsive 140 (all FR) |
| T21 | Phase 2 — SSO, object storage, malware scanning | BE+FE | L | P0 | Phase 2 | Planned | Institutional login, S3/R2/GCS, AV scan |
| T22 | Phase 3 — Redis, retention jobs, health/monitoring, formal a11y audit | Shared | L | P0 | Phase 3 | Planned | Multi-instance safe, retention, observability, WCAG audit |

**Feature coverage:** FR-01→T02; FR-02→T03; FR-03→T04; FR-04→T05; FR-05→T06; FR-06→T07; FR-07→T08; FR-08→T09; FR-09→T10; FR-10→T11; FR-11→T12; FR-12→T13; FR-13→T14; FR-14→T15; FR-15→T16; FR-16→T17; FR-17→T18; FR-18→T15; FR-19→T19; FR-20→T01.

## 6. API Endpoints (seeded from PRD §9)

| Endpoint (target) | Phase 1 path | Method | Auth | Milestone |
|---|---|---|---|---|
| `/api/v1/home` | `/api/home` | GET | Public | M1 |
| `/api/v1/site` | `/api/site` | GET | Public | M1 |
| `/api/v1/search` | `/api/search` | GET | Public | M3 |
| `/api/v1/services` | `/api/services` | GET | Public | M1 |
| `/api/v1/services/{slug}` | `/api/services/{slug}` | GET | Public | M1 |
| `/api/v1/services/{slug}/requests` | same | POST | Public | M2 |
| `/api/v1/platforms` | `/api/platforms` | GET | Public | M1 |
| `/api/v1/platforms/{slug}` | same | GET | Public | M1 |
| `/api/v1/documents` | `/api/documents` | GET | Public | M1 |
| `/api/v1/documents/{slug}` | same | GET | Public | M1 |
| `/api/v1/documents/{slug}/download` | same | GET | Public | M3 |
| `/api/v1/articles` | `/api/articles` | GET | Public | M1 |
| `/api/v1/articles/{slug}` | same | GET | Public | M1 |
| `/api/v1/pages/{slug}` | same | GET | Public | M1 |
| `/api/v1/team` | `/api/team` | GET | Public | M3 |
| `/api/v1/team/members/{id}/photo` | same | GET | Public | M3 |
| `/api/v1/track` | same | POST | Ticket+token | M2 |
| `/api/v1/track/respond` | same | POST | Ticket+token | M2 |
| `/api/v1/track/comments` | same | POST | Ticket+token | M2 |
| `/api/v1/track/attachments` | same | POST | Ticket+token | M2 |
| `/api/v1/requests/attachments/{id}` | same | GET | Ticket+token / staff | M2 |
| `/api/v1/auth/login` | same | POST | Public | M2 |
| `/api/v1/auth/logout` | same | POST | Session | M2 |
| `/api/v1/auth/session` | same | GET | Public | M2 |
| `/api/v1/auth/activate` | same | POST | Public | M2 |
| `/api/v1/admin/dashboard` | same | GET | requests.read | M3 |
| `/api/v1/admin/requests` | same | GET | requests.read | M3 |
| `/api/v1/admin/requests/options` | same | GET | requests.read | M3 |
| `/api/v1/admin/requests/{id}` | same | GET | requests.read | M3 |
| `/api/v1/admin/requests/{id}` | same | PATCH | requests.manage | M3 |
| `/api/v1/admin/requests/{id}/attachments` | same | POST | requests.manage | M3 |
| `/api/v1/admin/services` | same | GET, POST | services.manage | M3 |
| `/api/v1/admin/services/{id}` | same | GET, PATCH | services.manage | M3 |
| `/api/v1/admin/documents` | same | GET, POST | documents.manage | M3 |
| `/api/v1/admin/documents/options` | same | GET | documents.manage | M3 |
| `/api/v1/admin/documents/{id}` | same | GET, PATCH | documents.manage | M3 |
| `/api/v1/admin/documents/{id}/versions` | same | POST | documents.manage | M3 |
| `/api/v1/admin/documents/{id}/download` | same | GET | documents.manage | M3 |
| `/api/v1/admin/platforms` | same | GET, POST | content.manage | M3 |
| `/api/v1/admin/platforms/{id}` | same | PATCH, DELETE | content.manage | M3 |
| `/api/v1/admin/platform-categories` | same | GET, POST | content.manage | M3 |
| `/api/v1/admin/platform-categories/{id}` | same | PATCH, DELETE | content.manage | M3 |
| `/api/v1/admin/articles` | same | GET, POST | content.manage | M3 |
| `/api/v1/admin/articles/{id}` | same | GET, PATCH | content.manage | M3 |
| `/api/v1/admin/pages` | same | GET, POST | content.manage | M3 |
| `/api/v1/admin/pages/{id}` | same | PATCH | content.manage | M3 |
| `/api/v1/admin/team/members` | same | GET, POST | content.manage | M3 |
| `/api/v1/admin/team/members/{id}` | same | GET, PATCH, DELETE | content.manage | M3 |
| `/api/v1/admin/team/members/{id}/photo` | same | GET | content.manage | M3 |
| `/api/v1/admin/team/groups` | same | GET, POST | content.manage | M3 |
| `/api/v1/admin/team/groups/{id}` | same | PATCH, DELETE | content.manage | M3 |
| `/api/v1/admin/users` | same | GET, POST | users.manage | M3 |
| `/api/v1/admin/users/{id}` | same | PATCH | users.manage | M3 |
| `/api/v1/admin/users/{id}/resend` | same | POST | users.manage | M3 |
| `/api/v1/admin/roles` | same | GET | roles.manage | M3 |
| `/api/v1/admin/roles/{slug}` | same | PATCH | roles.manage | M3 |
| `/api/v1/admin/settings` | same | GET, PUT | settings.manage | M3 |
| `/api/v1/admin/notifications` | same | GET | notifications.read | M3 |
| `/api/v1/admin/audit` | same | GET | audit.read | M3 |
| `/sitemap.xml` | same | GET | Public | M3 |
| `/robots.txt` | same | GET | Public | M3 |

**Envelope:** `{ "data": {}, "message": "", "error": "" }` under `/api/v1/`. Phase 1 returns raw JSON at `/api/*`; field errors arrive as `data.errors` today and map to top-level `errors` in the target contract.

## 7. Data Model (seeded from PRD §10)

| Collection | Key fields | Indexes | Milestone |
|------------|------------|---------|-----------|
| `roles` | slug, label, permissions[], isSystem | slug unique | M2 |
| `permissions` | key, label, group, description | key unique | M2 |
| `users` | name, email, passwordHash, role, active, emailVerifiedAt, lastLoginAt, createdBy, isSeedData | email unique, role, active | M2 |
| `sessions` | tokenHash, user, expiresAt, ip, userAgent | tokenHash unique, user, TTL expiresAt | M2 |
| `activationtokens` | tokenHash, user, expiresAt, usedAt | tokenHash unique, user, TTL expiresAt | M2 |
| `services` | name, slug, descriptions, category, active, acceptsRequests, featured, order, arrays, currentFormVersionId, formVersionNumber, seo | slug unique, active, featured, order | M3 |
| `serviceformversions` | service, version, fields[], createdBy | (service, version) unique | M3 |
| `servicerequests` | ticketNumber, trackingTokenHash, requester*, service*, formVersion, formSnapshot, data, status, substatus, assignedTo, idempotencyKey, lastActivityAt | ticketNumber unique, status/service/user/email/assigned/lastActivity; idempotencyKey sparse unique; (user,createdAt), (status,createdAt), (service,createdAt) | M2 |
| `requeststatushistories` | request, fromStatus, toStatus, substatus, note, actor, actorName | request | M2 |
| `requestcomments` | request, author, authorName, message, visibility | request | M2 |
| `requestattachments` | request, fieldKey, originalName, storageKey, mimeType, size, uploadedBy | request, storageKey unique | M2 |
| `documents` | title, slug, category, description, published, archived, currentVersionId, services[] | slug unique, category, published, archived | M3 |
| `documentversions` | document, version, fileName, storageKey, mimeType, size, effectiveDate, changelog | (document,version) unique, storageKey unique | M3 |
| `teamgroups` | name, slug, order | slug unique, order | M3 |
| `teammembers` | name, position, unit, bio, group, order, published, photo*, focalX/Y | group, order, published | M3 |
| `platformcategories` | name, slug, description, order, active | slug unique, order | M3 |
| `digitalplatforms` | name, slug, description, category, categoryId, status, audiences, published, featured, modules, metrics, seo | slug unique, categoryId, published, featured, status | M3 |
| `articles` | title, slug, excerpt, content, category, author, publishedAt, status, seo | slug unique, status | M3 |
| `pages` | title, slug, excerpt, content, seo, published | slug unique | M3 |
| `sitesettings` | key, identity, contact, hero, seo, metrics, devSeedMode | key unique (`global`) | M1 |
| `notificationlogs` | template, recipient, status, provider, error, reference | — | M2 |
| `auditlogs` | actor, actorName, actorEmail, action, entityType, entityId, metadata, ip, userAgent | actor, action, entityType | M2 |

Full field-level schema is in `docs/DATA_CATALOG.md`.

## 8. Env Vars & Config (from PRD §12)

| Var | Dev | Staging | Prod | Secret | Required at startup |
|-----|-----|---------|------|--------|---------------------|
| `NUXT_MONGODB_URI` | `mongodb://127.0.0.1:27017/digital_unuy` | staging URI | prod URI | Yes | Yes |
| `NUXT_PUBLIC_SITE_URL` | `http://localhost:3000` | staging URL | production URL | No | Yes |
| `NUXT_SESSION_SECRET` | dev value | staging secret | prod secret | Yes | Yes (not used for session signing) |
| `NUXT_UPLOAD_DIR` | `.data/uploads` | persistent volume | persistent volume | No | Yes |
| `NUXT_SMTP_HOST` | empty (outbox) | staging SMTP | campus SMTP | No | Prod: yes for email |
| `NUXT_SMTP_PORT` | `587` | `587` | campus port | No | No |
| `NUXT_SMTP_SECURE` | `false` | per provider | per provider | No | No |
| `NUXT_SMTP_USER` / `NUXT_SMTP_PASS` | empty | staging | prod | Pass yes | No |
| `NUXT_EMAIL_FROM` | local sender | staging sender | official sender | No | No |
| `NUXT_PUBLIC_NOINDEX` | — | `true` | unset | No | No |
| `SEED_ADMIN_PASSWORD` / `SEED_PETUGAS_PASSWORD` | optional overrides | — | never | Yes | No |

Full matrix and `.env.example` guidance are in `docs/ENVIRONMENT_VARIABLES.md`.

## 9. Dependency Order & Milestones

```
M1 Public catalog
T01 (shell/tokens) → T02 (home) → T03 (services) → T15 (settings/site)

M2 Requests & auth
T02/T03 → T04 (form submit) → T05 (tracking/follow-up)
T01 → T06 (auth/activation) → T19 (security)

M3 Operations & CMS
T06/T08 → T07 (dashboard)
T04/T05 → T08 (lifecycle) → T17 (notifications)
T06/T19 → T14 (users/roles)
T03 → T09 (service admin/form builder)
T09 → T10 (documents) / T11 (ecosystem) / T12 (publications/pages) / T13 (team)
T02 → T18 (search)
T06 → T16 (audit)

M4 QA & hardening
T20 (e2e/visual/responsive)
→ Phase 2: T21 (SSO, object storage, malware scan)
→ Phase 3: T22 (Redis, retention, monitoring, a11y)
```

**Parallelization:** after T03/T06, CMS tasks T10–T13 can split across two engineers. T14 and T08 are the highest-risk paths. T20 runs after every milestone.

**First-milestone DoD:** T01–T03, T15 complete; public pages render with real settings; catalog search/filter works; build passes.

**Phase 1 DoD:** M1–M3 complete and T20 passed; all FRs traceable; E2E 105/105, visual 44/44, responsive 140/140; documented seed and limitations.

## 10. Definition of Done

**Per task**

- Acceptance criteria in §5 met and demonstrable.
- TypeScript strict passes; `npm run build` succeeds.
- Server-side validation and permission checks exist for every new input/route.
- Audit entry written for sensitive mutations.
- Responsive at 375px and 1440px; empty/error/loading states present.
- File handling uses MIME/extension/magic-byte validation and private storage.
- Docs updated when behavior/config changes.

**Global (Phase 1)**

- Public pages SSR without console errors.
- E2E, visual, and responsive scripts pass.
- `.env.example` current; no secret committed.
- Seed documented; `seed:reset` never run in production.
- Known limitations in README remain accurate.

## 11. Risks & Rollout

| Risk | Impact | Mitigation | Rollout |
|------|--------|------------|---------|
| Local uploads on one node | Data loss, no scaling | Persistent volume + backup; object storage Phase 2 | Single instance only |
| In-memory rate limit | Reset on restart, per process | Single instance; Redis Phase 3 | Do not scale out |
| SMTP unconfigured | No activation/notification email | Configure campus SMTP before go-live | Staging email test |
| Seed credentials | Unauthorized access | Never run seed in prod; rotate passwords | Pre-launch checklist |
| No malware scanning | Malicious upload | MIME/magic-byte validation; AV Phase 2 | Monitor uploads |
| MongoDB unavailable | Public/admin reads fail | Managed MongoDB/replica; backup | Restore drill |
| Migration/seed failure | Broken content | Backup before seed/migrate; staging first | Forward-only |
| Request status race | Wrong state | Conditional update + 409 | Monitor conflicts |
| Role permission cache | Up to 60s stale permissions | Document; clear cache on role update | — |
| Personal GitHub remote | SOP non-compliance | GitLab migration future | Before prod |
| No health endpoint | Slow incident detection | Add in Phase 3; external probe on `/` | — |

**Rollback:** keep the previous app image and MongoDB dump plus upload directory snapshot. Roll back the app first; restore DB/uploads only if a migration/seed changed data. Uploads and DB must be restored together.

## 12. Document Info & Sign-off

| Field | Value |
|-------|-------|
| Status | Draft |
| Version | 0.1 |
| Source PRD | PRD v1.0 — `docs/PRD.md` |
| Source Design Brief | Design Brief v0.1 — `docs/DESIGN_BRIEF.md` |
| Author / Owner | Arief Faqihudin |
| Approver + date | Belum ditandatangani |
| Issue key | Belum ditetapkan (Q3 di PRD) |
| File location | `docs/IMPLEMENTATION_PLAN.md` |

**Assumptions**

- A1: Phase 1 tetap monolit Nuxt dan tetap menjadi referensi perilaku. Target arsitektur adalah split Go BE + Nuxt FE (D-STACK); rencana migrasinya ada di §13, bukan asumsi.
- A2: Deployment Phase 1 adalah satu instance Node dengan MongoDB dan upload lokal.
- A3: Domain publik dan SMTP ditetapkan operator.
- A4: `/api/v1` + envelope diterapkan pada Phase 2/rewrite; Phase 1 memakai `/api/*` raw.
- A5: Stack tidak lagi dipilih per proyek. Go + Gin untuk BE dan Nuxt untuk FE berlaku untuk proyek baru maupun pemisahan repo.

**Decisions**

- **D-STACK (2026-11-07, D8 di PRD §16):** Semua proyek memakai Go untuk backend dan Vue/Nuxt untuk frontend. Untuk Digital UNUY ini berarti dua repo (`fe/`, `be/`) dan menutup pertanyaan "apakah backend Go dikerjakan" — yang tersisa hanyalah urutan migrasi dan model auth (§13).

**Open questions**

- Q1: Target waktu penyelesaian dan performa publik.
- Q2: Domain, DNS, TLS, dan operator VM.
- Q3: Nomor issue dan tanggal persetujuan.
- Q4: SMTP kampus dan alamat pengirim resmi.
- Q5: Backup/restore dan uji pemulihan.
- Q6: Rencana SSO/integrasi institusional dan object storage.
- Q7: Multi-instance/Redis.
- Q8: Retensi permohonan, audit, dan lampiran.
- Q9: Audit aksesibilitas formal.
- Q-API: Migrasi `/api/*` ke `/api/v1` + envelope.

## 13. Migration to Go + Nuxt (target)

**Principle — strangler, not big bang.** The FE keeps serving and owns the public origin; endpoints move to the Go service behind the FE BFF one domain at a time. A monolith route is deleted only after its Go replacement passes parity checks, so the app is deployable at every step.

**Prerequisites (must be settled before the first endpoint moves):**

1. **Auth transport** (§4.1) — BFF with an internal token, or cross-site cookie + CORS. Every endpoint depends on this; porting first and deciding later would double the work.
2. **Contract freeze per wave** — the endpoints in scope are updated in `API_CONTRACTS.md` to `/api/v1` + envelope before implementation starts.
3. **Index parity** — MongoDB indexes required by the ported queries are captured in `be/migrations/` and checked against `DATA_CATALOG.md`.
4. **House scaffolding** — Gin router, swaggo (`make swagger`), centralized `httperr`, async activity log, and config validation exist before domain code.

**Porting order (by contract section, not by file):**

| Wave | Scope | Contract sections | Depends on |
|---|---|---|---|
| 0 | Skeleton: health, `auth/me`, settings read, BFF wiring | §3 (partial), §9 (settings read) | Auth transport decision |
| 1 | Public content: services, ecosystem, publications, pages, team, SEO routes | §1, §8 (public reads), §10 | Wave 0 |
| 2 | Request submission, tracking, requester follow-up, attachments | §2 | Wave 0; upload handling decision |
| 3 | Admin reads: dashboard, requests, services, documents, ecosystem, publications | §4–§8 (reads) | Wave 1 |
| 4 | Admin writes + users, roles, settings, notifications, audit | §4–§9 (writes) | Wave 3; activity log live |
| 5 | Uploads, images, email | §6, §8 (files), notifications | Object storage (T21) or BFF-proxied uploads |

**Per-wave Definition of Done:**

- Every ported endpoint matches `API_CONTRACTS.md` (status codes, envelope, `404` vs `403` semantics).
- Swagger annotations present and `make swagger` clean; activity log emitted for every state-changing action.
- FE calls the endpoint through the BFF only — no direct BE URL in the browser.
- Contract tests or E2E cover the ported flow; `go vet` and `go test ./...` pass.
- The corresponding Nuxt route is deleted in the same change, not left as a fallback.

**Cutover & rollback:** keep the monolith deployable until wave 5 is complete. If a ported endpoint regresses, restore the Nuxt route from the previous release tag; the Go route stays deployed but unreferenced. Data stays in one MongoDB database for the whole migration, so no data migration is required — only index additions.

**What does not change:** RBAC semantics (11 permission keys, last-superadmin guard), the MongoDB collections and field names, the FE screens and design tokens, SEO policy, and the audit trail requirements. The split is a transport and process change, not a product change.

**Open questions specific to the migration:**

- **M1:** BFF vs cross-site cookie (blocks wave 0).
- **M2:** Uploads stay behind the FE BFF, or move to object storage as part of the split (affects wave 2 and 5).
- **M3:** Rate limiting moves to a shared store (Redis, T22) or stays per service instance.
- **M4:** Whether `/api/v1` + envelope applies retroactively to endpoints still served by the monolith during the transition (currently A4 says Phase 2/rewrite).
