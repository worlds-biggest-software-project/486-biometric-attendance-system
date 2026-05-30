# Biometric Attendance System — Phased Development Plan

> Project: 486-biometric-attendance-system · Created: 2026-05-31
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. It defines a privacy-first, self-hostable, mobile-first biometric time-and-attendance platform with AI-native anomaly detection. The MVP (Phases 1–7) ships mobile biometric clock-in, geofencing, shift/overtime rules, payroll export, RBAC, consent management, offline sync, and an immutable audit log. Phases 8–11 add hardware terminal adapters, ML anomaly detection, multi-jurisdiction compliance automation, and natural-language reporting.

---

## Core Requirements (synthesised)

1. **What it does** — Binds every clock event to a verified individual via facial/fingerprint biometrics, eliminating buddy punching, then automates the downstream shift, overtime, leave, payroll, and compliance workflows. Biometric matching happens on-device by default; the backend never stores raw biometric images, only irreversible cancellable templates.
2. **Who uses it** — HR administrators, payroll administrators, operations/site managers, and employees (self-service). Plus a system-admin/operator persona for self-hosted deployments.
3. **Key differentiators** — Privacy-by-construction (cancellable biometrics per ISO/IEC 24745, automated consent + deletion), AI-native anomaly detection (ML not static thresholds), device-agnostic integration (ZKTeco/Suprema/eSSL/mobile under one API), and open-source with SaaS polish.
4. **MVP feature set** — Mobile face + fingerprint clock-in, GPS geofencing, fixed-shift scheduling with overtime, manager exception dashboard, payroll CSV + Xero/QuickBooks connectors, RBAC (employee/manager/HR admin), enrolment consent + opt-out, offline sync, immutable audit log.
5. **Post-MVP** — Hardware terminal adapters, ML anomaly detection, multi-jurisdiction consent templates + automated retention/deletion, leave/PTO, self-service portal, shift swap/roster with policy validation, NL reporting (LLM), predictive absence, touchless multi-modal.
6. **Deployment model** — Self-hosted on-prem (primary), cloud SaaS, and hybrid (cloud control plane + on-prem biometric processing). Docker Compose is the reference deployment. Multi-tenant by organisation.
7. **Integration surface** — Payroll/HRMS (Xero, QuickBooks, ADP, Workday, SAP SuccessFactors), hardware terminals (ZKTeco ZKBioTime, Suprema BioStar X, eSSL), SSO IdPs (Azure AD, Okta, Google) via OIDC, LLM providers (anomaly explanations, NL reporting), optional MCP server.
8. **Standards compliance** — ISO/IEC 24745 (template protection: irreversibility, unlinkability, revocability), ISO/IEC 30107-3 (PAD/liveness levels), ISO/IEC 19794 + CBEFF (template interchange), OpenAPI 3.1.1 + JSON Schema 2020-12 (API), OAuth 2.0 / RFC 7519 JWT / OIDC (auth), WebAuthn/FIDO2 (self-service portal login), TLS 1.3 (transport), ISO 8601 (timestamps), iCalendar RFC 5545 (schedule export), OWASP API Security Top 10 (2023), BIPA / GDPR Art. 9 / CPRA (privacy), MCP (AI integration).
9. **Data model** — **Suggestion 4 (TimescaleDB + relational hybrid)** is adopted as the primary model: relational reference tables for organisation/employee/config data, TimescaleDB hypertables for clock events / heartbeats / anomaly scores / audit log, continuous aggregates for dashboards, and native retention policies for compliance. JSONB columns from Suggestion 3 are incorporated for jurisdiction-, vendor-, and modality-specific variability. The audit_log hypertable doubles as the immutable compliance record (append-only).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary backend language | **Python 3.12** | The differentiators (ML anomaly detection, liveness/PAD, NL reporting, face/fingerprint matching) are all Python-ecosystem strengths. One language covers API, workers, and ML. |
| API framework | **FastAPI** | Native OpenAPI 3.1 generation (a stated standard), Pydantic v2 validation maps directly to JSON Schema 2020-12, async for webhook/sync workloads, first-class dependency injection for RBAC. |
| Data validation | **Pydantic v2** | Validates all request/response bodies and JSONB payloads; serialises to JSON Schema 2020-12 used by the OpenAPI spec. |
| Database | **PostgreSQL 16 + TimescaleDB 2.x** | Adopts data-model Suggestion 4. Time-series is the dominant pattern (clock events, heartbeats, anomaly scores). Hypertables give automatic partitioning, 10–20x compression for multi-year retention, continuous aggregates for dashboards, and retention policies that directly implement GDPR/BIPA deletion. Full SQL/JSONB/FK support retained. |
| ORM / migrations | **SQLAlchemy 2.0 + Alembic** | Mature, supports raw SQL for hypertable creation and continuous aggregates that the ORM cannot model. Alembic versions both relational tables and Timescale DDL. |
| Task queue | **Celery + Redis** | Async workloads: payroll export jobs, offline-sync ingestion, anomaly-detection batches, retention/deletion sweeps, webhook delivery with retries. Redis also serves rate limiting and caching. |
| Secrets / key management | **HashiCorp Vault (Transit engine) with env-var fallback** | Per-device / per-org encryption keys for cancellable biometric templates (ISO 24745). Transit engine performs envelope encryption so raw keys never leave Vault. Local dev falls back to a file-backed key provider. |
| Biometric matching (server-side fallback) | **face_recognition (dlib) for face, SourceAFIS (via JPype) for fingerprint** | Open-source, no per-match licensing. Primary matching is on-device; server-side matching is a fallback for webcam/kiosk enrolment and verification. Template format follows ISO/IEC 19794 where the engine supports it. |
| Liveness / PAD | **MediaPipe + a lightweight CNN (ONNX Runtime)** for passive 2D liveness; pluggable interface for device-reported 3D/IR depth | ISO/IEC 30107-3 framing. On-device 3D/IR liveness is trusted when reported; server-side passive liveness is the mobile-camera fallback. Self-improving model retrains from flagged attempts. |
| ML anomaly detection | **scikit-learn (IsolationForest, DBSCAN) + statsmodels for trend/drift** | Unsupervised detection of buddy-punch clusters (DBSCAN on co-location/time), overtime spikes (IsolationForest), and schedule drift (linear trend on clock-in times). No labelled fraud data required at launch. |
| LLM integration | **Provider-agnostic via an `LLMClient` abstraction (Anthropic + OpenAI adapters)** | NL reporting and anomaly explanations. Read-only, parameterised SQL generation guarded by an allowlist. Exposed additionally as an MCP server. |
| Mobile app | **React Native (Expo) + expo-local-authentication + expo-camera + expo-location** | Single codebase for iOS/Android. Uses device Face ID / fingerprint and on-device camera for capture; never transmits raw biometric images by default. Offline queue via SQLite + WatermelonDB. |
| Web frontend | **Next.js 15 (App Router) + TypeScript + shadcn/ui + TanStack Query** | Admin/manager dashboards, HR console, employee self-service portal, consent workflows. Server components for data-heavy dashboards; WebAuthn for passwordless login. |
| Auth | **OAuth 2.0 + JWT (RFC 7519) + OIDC; WebAuthn/FIDO2 for portal** | OIDC SSO with Azure AD/Okta/Google for staff; WebAuthn passkeys for employee self-service (NIST 800-63-4 AAL2). Bearer tokens (RFC 6750) for all API and integration calls. |
| Containerisation | **Docker + Docker Compose (reference); Helm chart (v1.1)** | Self-hosted is the primary deployment mode. Compose bundles API, workers, Postgres/Timescale, Redis, Vault, web. |
| Testing | **pytest + pytest-asyncio + testcontainers** (backend); **Vitest + Playwright** (web); **Jest + Detox** (mobile) | testcontainers spins real Postgres/Timescale/Redis for integration tests. Playwright/Detox for E2E. |
| Code quality | **ruff (lint+format), mypy (strict), bandit (security)** backend; **eslint + prettier + tsc** frontend | bandit covers OWASP-relevant static checks on a security-critical codebase. |
| Package managers | **uv (Python), pnpm (web), npm/Expo (mobile)** | Fast, reproducible installs. |
| API docs | **Swagger UI + Redoc from the FastAPI OpenAPI 3.1 spec** | Published spec is a stated standard; used by HRMS integrators and hardware-adapter authors. |
| Observability | **structlog (JSON logs) + Prometheus metrics + OpenTelemetry traces** | Device health monitoring, sync lag, match-score drift, and audit completeness. |

### Project Structure

```
biometric-attendance-system/
├── README.md
├── LICENSE                          # AGPLv3 (matches TimeTrex precedent; copyleft for self-hosted core)
├── docker-compose.yml               # api, worker, beat, postgres+timescale, redis, vault, web
├── docker-compose.dev.yml
├── .env.example
├── Makefile                         # make up / test / lint / migrate / seed
├── openapi/                         # exported, version-pinned OpenAPI 3.1 spec snapshots
│
├── backend/
│   ├── pyproject.toml               # uv-managed; ruff, mypy, bandit config
│   ├── alembic.ini
│   ├── migrations/                  # Alembic; includes raw-SQL Timescale ops
│   ├── src/
│   │   └── bas/                     # "Biometric Attendance System"
│   │       ├── main.py              # FastAPI app factory, router mounting, middleware
│   │       ├── config.py            # Pydantic Settings (env-driven)
│   │       ├── db/
│   │       │   ├── base.py          # SQLAlchemy engine/session, RLS session vars
│   │       │   ├── models/          # relational ORM models
│   │       │   └── timescale.py     # hypertable / continuous-aggregate helpers
│   │       ├── schemas/             # Pydantic request/response models (→ JSON Schema)
│   │       ├── api/
│   │       │   ├── deps.py          # auth, RBAC, tenant, pagination dependencies
│   │       │   ├── v1/
│   │       │   │   ├── auth.py
│   │       │   │   ├── organisations.py
│   │       │   │   ├── employees.py
│   │       │   │   ├── enrolment.py        # consent + template enrolment
│   │       │   │   ├── clock.py            # clock in/out, verification
│   │       │   │   ├── sync.py             # offline batch ingestion
│   │       │   │   ├── shifts.py
│   │       │   │   ├── attendance.py       # daily summaries, exceptions
│   │       │   │   ├── leave.py
│   │       │   │   ├── payroll.py
│   │       │   │   ├── devices.py
│   │       │   │   ├── anomalies.py
│   │       │   │   ├── compliance.py       # consent templates, deletion, DSAR
│   │       │   │   ├── reporting.py        # NL query endpoint
│   │       │   │   └── webhooks.py         # device + integration inbound
│   │       ├── domain/
│   │       │   ├── biometrics/      # template transform, matching, liveness, key mgmt
│   │       │   ├── attendance/      # work-rule engine, overtime, rounding, daily rollup
│   │       │   ├── scheduling/      # shift resolution, roster, swap policy
│   │       │   ├── leave/           # accrual, balance, request workflow
│   │       │   ├── anomaly/         # detectors + model registry
│   │       │   ├── compliance/      # jurisdiction rules, retention, consent engine
│   │       │   └── payroll/         # exporters + connector mapping
│   │       ├── integrations/
│   │       │   ├── devices/         # base adapter + zkteco, suprema, essl, mobile
│   │       │   ├── payroll/         # base connector + xero, quickbooks, adp, csv
│   │       │   ├── llm/             # LLMClient + anthropic/openai adapters
│   │       │   └── idp/             # OIDC clients
│   │       ├── workers/             # Celery tasks: sync, export, anomaly, retention
│   │       ├── security/            # crypto, RBAC policy, audit writer, rate limit
│   │       └── mcp/                 # read-only MCP server over reporting API
│   └── tests/
│       ├── unit/
│       ├── integration/             # testcontainers-backed
│       ├── e2e/
│       └── fixtures/                # sample diffs, templates, payloads, vectors
│
├── web/                             # Next.js admin + self-service
│   ├── package.json
│   ├── app/
│   ├── components/
│   ├── lib/                         # generated API client from openapi spec
│   └── tests/                       # Vitest + Playwright
│
└── mobile/                          # React Native (Expo)
    ├── package.json
    ├── app/
    ├── src/
    │   ├── biometric/               # capture, on-device match, liveness
    │   ├── offline/                 # SQLite queue + sync engine
    │   └── api/                     # generated client
    └── tests/                       # Jest + Detox
```

---

## Phase 1: Foundation, Tenancy & Auth

### Purpose
Establish the project skeleton, database with TimescaleDB, multi-tenant data isolation, authentication, RBAC, and the immutable audit log. Everything downstream depends on a tenant-scoped, authenticated, audited request context. After this phase the API boots, a user can authenticate, and every privileged action is recorded immutably.

### Tasks

#### 1.1 — Project scaffold, config, and Docker Compose

**What**: Bootable FastAPI app, Postgres+TimescaleDB, Redis, Vault, and web placeholder via Docker Compose, with environment-driven config.

**Design**:
- `bas.config.Settings` (Pydantic `BaseSettings`):
```python
class Settings(BaseSettings):
    environment: Literal["dev", "staging", "prod"] = "dev"
    database_url: PostgresDsn
    redis_url: RedisDsn
    vault_addr: str | None = None
    vault_token: SecretStr | None = None
    jwt_secret: SecretStr                      # HS256 dev; RS256 keys in prod
    jwt_algorithm: str = "RS256"
    access_token_ttl_minutes: int = 15
    refresh_token_ttl_days: int = 14
    default_jurisdiction: str = "gdpr"
    cors_origins: list[str] = []
    model_config = SettingsConfigDict(env_prefix="BAS_", env_file=".env")
```
- `bas.main.create_app()` mounts `/api/v1`, `/health`, `/metrics`, Swagger at `/docs`, Redoc at `/redoc`.
- `docker-compose.yml` services: `timescale/timescaledb:2.x-pg16`, `redis:7`, `hashicorp/vault`, `api`, `web`. Healthchecks gate `api` start on DB readiness.
- `/health` returns `{status, db: bool, redis: bool, vault: bool, version}`.

**Testing**:
- `Unit: Settings loads from env with BAS_ prefix → correct typed values; missing jwt_secret → ValidationError`.
- `Integration: docker compose up → GET /health returns 200 with db=true, redis=true`.
- `Unit: GET /docs serves OpenAPI 3.1 JSON with info.version set`.

#### 1.2 — Database bootstrap, TimescaleDB, and Alembic baseline

**What**: Alembic migration creating the extension, relational reference tables, and the first hypertables.

**Design**:
- Migration `0001_baseline`: `CREATE EXTENSION IF NOT EXISTS timescaledb;` then relational tables `organisations`, `sites`, `departments`, `employees`, `users` (auth identities, separate from `employees`), `roles`, `user_roles`.
- `users` links optionally to an `employee_id`; an HR admin may have no employee record.
```sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id) ON DELETE CASCADE,
    employee_id UUID REFERENCES employees(id) ON DELETE SET NULL,
    email VARCHAR(255) NOT NULL,
    password_hash VARCHAR(255),          -- NULL when SSO-only
    sso_subject VARCHAR(255),            -- OIDC sub
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
CREATE TABLE roles (id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(32) NOT NULL UNIQUE);   -- 'employee','manager','hr_admin','sys_admin'
CREATE TABLE user_roles (user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID REFERENCES roles(id) ON DELETE CASCADE,
    scope_site_id UUID REFERENCES sites(id),  -- manager scoped to a site
    PRIMARY KEY (user_id, role_id, scope_site_id));
```
- The `audit_log` hypertable from Suggestion 4 is created here with append-only rules and compression.
- `bas.db.timescale.create_hypertable(table, time_col, interval)` helper wraps raw SQL for migrations.
- Adopt the relational tables verbatim from data-model Suggestion 4; `employees.profile`, `organisations.settings`, `sites` geo fields included.

**Testing**:
- `Integration (testcontainers): run alembic upgrade head → timescaledb extension present, audit_log is a hypertable (query timescaledb_information.hypertables)`.
- `Integration: INSERT then UPDATE audit_log → row unchanged (append-only rule)`.
- `Integration: alembic downgrade base then upgrade head → idempotent`.

#### 1.3 — Multi-tenant isolation via Row-Level Security

**What**: PostgreSQL RLS enforcing organisation-scoped access on every tenant table, set per request.

**Design**:
- Each tenant table gets: `ALTER TABLE x ENABLE ROW LEVEL SECURITY;` and a policy `USING (organisation_id = current_setting('bas.org_id')::uuid)`.
- `bas.db.base.get_session()` dependency executes `SET LOCAL bas.org_id = :org` from the JWT claim at the start of each request transaction.
- A privileged `bas_migrator` role bypasses RLS; the app connects as a non-bypass role.
- Data-residency: `organisations.data_residency` drives which DB connection pool is used in multi-region deployments (single-region for MVP, interface stubbed).

**Testing**:
- `Integration: two orgs, query employees as org A session → only org A rows returned`.
- `Integration: attempt to SELECT another org's employee by id → 0 rows (RLS), API returns 404 not 403 (avoid existence leak, OWASP BOLA)`.
- `Unit: get_session sets bas.org_id from request state`.

#### 1.4 — Authentication: JWT, refresh, OIDC SSO

**What**: Password and OIDC login issuing short-lived access JWTs and refresh tokens.

**Design**:
- Endpoints:
  - `POST /api/v1/auth/login` `{email, password}` → `{access_token, refresh_token, expires_in}`.
  - `POST /api/v1/auth/refresh` `{refresh_token}` → new access token.
  - `GET /api/v1/auth/oidc/{provider}/start` → 302 to IdP; `GET /api/v1/auth/oidc/{provider}/callback` → tokens.
  - `POST /api/v1/auth/logout` → revokes refresh token (Redis denylist).
- JWT claims: `sub` (user id), `org` (organisation id), `roles` (list), `scopes` (site ids), `exp`, `iat`, `jti`. RS256 in prod.
- Passwords hashed with Argon2id. OIDC via `authlib`; `sso_subject` matched/provisioned just-in-time.
- All tokens are bearer (RFC 6750); enforce TLS at the proxy.

**Testing**:
- `Unit: valid credentials → JWT with correct org/roles claims; wrong password → 401`.
- `Unit: expired access token → 401; valid refresh → new access token; revoked refresh (jti in denylist) → 401`.
- `Integration (mocked IdP): OIDC callback with valid code → user provisioned, tokens issued`.
- `Unit: Argon2id verify round-trips; rehash on parameter upgrade`.

#### 1.5 — RBAC dependency and audit writer

**What**: Reusable FastAPI dependencies for role/scope checks, and a central audit writer used by all mutating endpoints.

**Design**:
- `require_roles(*roles)` and `require_scope_for_site(site_id_param)` dependencies read JWT claims.
- Role matrix: `employee` (own records), `manager` (site-scoped read + approvals), `hr_admin` (org-wide), `sys_admin` (config/devices/integrations).
- `audit.record(action, entity_type, entity_id, old, new, actor, context)` writes one append-only row; called via a FastAPI dependency `Audited` that captures actor/IP/user-agent automatically.
```python
async def record(action: str, entity_type: str, entity_id: UUID | None,
                 *, old: dict | None, new: dict | None, ctx: AuditContext) -> None: ...
```

**Testing**:
- `Unit: manager scoped to site A calls site-B endpoint → 403`.
- `Unit: employee calls hr_admin endpoint → 403`.
- `Integration: any successful mutation writes exactly one audit_log row with actor_id, ip, action`.
- `Integration: failed (rejected) mutation writes an audit row with outcome=denied`.

---

## Phase 2: Biometric Enrolment, Consent & Template Security

### Purpose
Implement the privacy core: consent capture before enrolment, irreversible cancellable biometric templates (ISO/IEC 24745), per-device/org key management, and opt-out handling. No clock event can reference a template that lacks active consent. After this phase an employee can be enrolled (or opt out) in a fully compliant, auditable way.

### Tasks

#### 2.1 — Consent capture workflow

**What**: Endpoints to present a jurisdiction-specific consent text, capture explicit consent, and revoke it.

**Design**:
- Adopt `biometric_consents` from Suggestion 4 (`consent_details` JSONB carries jurisdiction-specific fields per Suggestion 3 examples).
- Endpoints:
  - `GET /api/v1/enrolment/consent-template?jurisdiction=gdpr&modality=facial` → `{text, version, required_fields}`.
  - `POST /api/v1/enrolment/consent` `{employee_id, consent_type, jurisdiction, consent_details, consent_text_hash}` → consent record. Records `ip_address`, computes `deletion_due_at = granted_at + retention_days`.
  - `POST /api/v1/enrolment/consent/{id}/revoke` → sets `revoked_at`, enqueues template deactivation + deletion job.
- `consent_text_hash` is the SHA-256 of the exact text shown; the server re-hashes the stored template text and rejects mismatches (proof of informed consent for BIPA/GDPR Art. 9).
- Opt-out: revoking the only consent flips the employee to `alternative_verification` (PIN + photo) in `employees.profile`.

**Testing**:
- `Unit: consent with valid hash matching stored template → 201; mismatched hash → 422`.
- `Unit: deletion_due_at computed from retention_days`.
- `Integration: revoke consent → template deactivated, deletion job enqueued, audit row written`.
- `Unit (jurisdiction): GDPR template requires legal_basis + dpo_contact; missing → 422. BIPA requires written_release_obtained → 422 if false`.

#### 2.2 — Cancellable template transform & key management

**What**: Transform a raw biometric feature vector into an irreversible, revocable template, encrypted with a managed key.

**Design**:
- ISO/IEC 24745 properties: **irreversibility**, **unlinkability**, **revocability**.
- Transform: BioHashing-style — project the feature vector with a random orthonormal matrix seeded from `transformation_salt`, quantise, then HMAC. Re-enrolment uses a new salt → a new, uncorrelated template (revocability + unlinkability).
```python
class TemplateTransformer:
    def transform(self, features: np.ndarray, salt: bytes) -> bytes: ...
    def is_match(self, probe: bytes, stored: bytes, threshold: float) -> tuple[bool, float]: ...
```
- Encryption: `KeyProvider` interface; `VaultKeyProvider` uses Vault Transit (`encrypt`/`decrypt` by `encryption_key_ref`), `LocalKeyProvider` uses an env master key for dev. Templates stored as `template_blob BYTEA` (ciphertext). Raw images are never persisted.
- `biometric_templates` table from Suggestion 4 (`template_metadata` JSONB per modality).

**Testing**:
- `Unit: transform is deterministic for same (features, salt); different salt → uncorrelated output (cosine < 0.2)`.
- `Unit: irreversibility — given template + salt, no inverse recovers features within tolerance (statistical check)`.
- `Unit: is_match — same identity probe matches above threshold; different identity below`.
- `Integration (mocked Vault): enrol → ciphertext stored; decrypt round-trips; key_ref recorded`.
- `Security: API response for a template never contains template_blob (OWASP excessive data exposure)`.

#### 2.3 — Enrolment endpoint and quality gating

**What**: Enrol a template for an employee, gated on active consent and capture quality.

**Design**:
- `POST /api/v1/enrolment/templates` `{employee_id, consent_id, modality, features|template_payload, device_id, quality_score, metadata}`.
- Validations: active (non-revoked) consent of matching `consent_type` exists; `quality_score >= configured_min` (default 0.6); employee active.
- For mobile on-device enrolment the client sends the already-transformed template + salt; for server-side (webcam) it sends features and the server transforms.
- Re-enrolment: `POST /api/v1/enrolment/templates/{id}/reenrol` deactivates the old template and creates a new one with a fresh salt (cancellable biometrics).

**Testing**:
- `Unit: enrol without active consent → 409`.
- `Unit: quality below threshold → 422 with quality value in message`.
- `Integration: enrol then reenrol → old is_active=false, new is_active=true, distinct salt`.
- `Integration: enrol writes audit row (action=template_enrolled), no raw image stored anywhere`.

---

## Phase 3: Clock Events — Verification, Geofencing & Offline Sync

### Purpose
Deliver the heart of the product: recording a verified clock event. A clock-in must pass biometric match, liveness, and (for mobile) geofence checks, and offline-captured events must sync idempotently on reconnect. After this phase a verified employee can clock in/out online or offline, and the events land immutably in the time-series store.

### Tasks

#### 3.1 — Clock event ingestion and verification pipeline

**What**: Endpoint and pipeline that validates and records a single clock event.

**Design**:
- `clock_events` hypertable from Suggestion 4 (weekly chunks, compression after 30 days, `verification_meta` JSONB, `match_score`, `liveness_passed`, geo fields, `anomaly_flags`).
- `POST /api/v1/clock` body:
```json
{ "employee_id": "...", "device_id": "...", "event_type": "clock_in",
  "event_time": "2026-05-31T08:00:00Z", "verification_method": "facial",
  "match_score": 0.97, "liveness_passed": true, "liveness_method": "ir_depth",
  "location": {"latitude": 51.5, "longitude": -0.12, "accuracy_m": 5.0},
  "idempotency_key": "uuid", "template_id": "..." }
```
- Pipeline steps (each appends to `verification_meta`): resolve employee → check active → verify match_score ≥ method threshold → liveness check (trust device-reported 3D/IR; run server passive liveness for plain mobile selfies) → geofence check (3.2) → derive `event_type` validity (no double clock-in) → persist.
- Rejections return `409` with a structured reason and still write an audit + a rejected clock attempt row (for fraud analysis).
- ISO 8601 timestamps throughout.

**Testing**:
- `Unit: match_score below method threshold → rejected (reason=low_match)`.
- `Unit: liveness_passed=false → rejected (reason=liveness_failed)`.
- `Unit: clock_in when last event was clock_in → rejected (reason=invalid_sequence)`.
- `Integration: valid clock_in → row in clock_events hypertable, audit row, 201`.
- `Fixture: replay a known good face vector pair → match true; spoof vector → liveness false`.

#### 3.2 — Geofencing

**What**: Validate mobile clock-ins against site geofences.

**Design**:
- Haversine distance from event lat/lng to the employee's assigned `site` centre; `geo_fence_ok = distance <= site.geo_fence_radius_m`.
- Polygon geofences supported via `sites.location.geo_fence_polygon` (point-in-polygon) when present, else radius.
- Configurable enforcement: `block` (reject outside fence) vs `flag` (record `geo_fence_ok=false`, allow, mark for review) per site.
- `accuracy_m` over a threshold (default 50m) downgrades to `flag` to avoid false rejects from poor GPS.

**Testing**:
- `Unit: point inside radius → ok=true; outside → ok=false`.
- `Unit: point inside polygon → ok=true; outside → false`.
- `Unit: enforcement=block + outside → clock rejected; enforcement=flag → recorded with geo_fence_ok=false`.
- `Unit: accuracy_m=120 → downgraded to flag regardless of distance`.

#### 3.3 — Offline batch sync

**What**: Idempotent batch ingestion of events queued on a device while offline.

**Design**:
- `POST /api/v1/sync/events` `{device_id, events: [ClockEventIn...]}` processed by a Celery task for large batches.
- Idempotency: unique `(device_id, idempotency_key)` enforced via a Redis set + a unique partial index; duplicates are acknowledged, not re-inserted.
- Each event runs the full 3.1 pipeline; `synced_at = now()`, `source` preserved as `device`/`mobile`.
- Response: per-event status array `[{idempotency_key, status: accepted|duplicate|rejected, reason?}]` so the client can clear its local queue precisely.
- Out-of-order tolerance: events are sorted by `event_time` before sequence validation; clock skew beyond a configurable window flags rather than rejects.

**Testing**:
- `Integration: submit batch of 50 → all accepted; resubmit same batch → all duplicate, no new rows`.
- `Integration: batch with one invalid-sequence event → that one rejected, others accepted`.
- `Unit: events arrive out of order → sorted by event_time before validation`.
- `Integration: device offline 3 days, syncs 200 events → all land with correct synced_at and original event_time`.

---

## Phase 4: Scheduling, Work Rules & Attendance Computation

### Purpose
Turn raw clock events into meaningful attendance: shift patterns, the work-rule/overtime engine, and the daily attendance rollup driven by TimescaleDB continuous aggregates. After this phase managers see who is present, late, or absent, with overtime computed.

### Tasks

#### 4.1 — Shift patterns and assignments

**What**: Define shift patterns (fixed for MVP, rotating-ready) and assign them to employees over date ranges.

**Design**:
- `shift_patterns.rules` JSONB per Suggestion 3 (`type: fixed|weekly_rotating`, per-day start/end/break). `employee_shift_assignments` with `effective_from/to` and `overrides` JSONB.
- `scheduling.resolve_shift(employee_id, date) -> ResolvedShift | None` picks the active assignment, applies rotation week math and per-date overrides.
```python
@dataclass(frozen=True)
class ResolvedShift:
    start: time; end: time; break_minutes: int; is_overnight: bool; off: bool
```
- Endpoints: CRUD `/api/v1/shifts/patterns`, `/api/v1/shifts/assignments`; `GET /api/v1/shifts/resolve?employee_id&date`.
- iCalendar (RFC 5545) export: `GET /api/v1/shifts/calendar.ics?employee_id` for Google/Outlook/Apple.

**Testing**:
- `Unit: fixed pattern resolves correct start/end for a weekday; weekend off → off=true`.
- `Unit: weekly_rotating week-2 resolves the alternate schedule`.
- `Unit: per-date override replaces resolved times; off-override → off=true`.
- `Unit: .ics export validates against RFC 5545 (parseable by icalendar lib)`.

#### 4.2 — Work-rule & overtime engine

**What**: Pure functions computing worked minutes, overtime, rounding, and grace from a day's events plus the resolved shift and work rule.

**Design**:
- `work_rules.rules` JSONB per Suggestion 3 (daily/weekly thresholds, OT multipliers, grace, rounding, auto-clock-out, min break).
- `attendance.compute_day(events, shift, rule) -> DayComputation`:
```python
@dataclass
class DayComputation:
    first_in: datetime | None; last_out: datetime | None
    work_minutes: int; break_minutes: int; overtime_minutes: int
    status: Literal['present','absent','late','half_day','leave','holiday','weekend']
    exceptions: list[str]            # 'late_arrival','early_departure','missed_punch'
```
- Logic: pair clock_in/out (and break_start/end), apply rounding (`nearest_15` etc.), subtract breaks, compare against thresholds for OT, apply grace to lateness, detect missed punches (odd number of events) and auto-clock-out after configured hours.

**Testing**:
- `Unit: 9:00–17:00 with 60m break, 8h threshold → 420 work mins, 0 OT, status present`.
- `Unit: arrival 9:07 with 5m grace → late_arrival; arrival 9:03 → present`.
- `Unit: 10h worked, 8h threshold → 120 OT minutes`.
- `Unit: clock_in with no clock_out → missed_punch exception, auto-clock-out applied after N hours`.
- `Unit: rounding nearest_15 — 8:52 in rounds to 9:00`.

#### 4.3 — Daily attendance rollup via continuous aggregates + reconciliation

**What**: Maintain `daily_attendance` using the Timescale continuous aggregate plus a reconciliation worker that applies work rules.

**Design**:
- Use Suggestion 4's `daily_attendance_summary` continuous aggregate for raw first-in/last-out/counts (auto-refreshed hourly).
- A Celery task `reconcile_day(employee_id, date)` reads the aggregate + events, runs `compute_day`, and upserts the relational `daily_attendance` row (with `status`, OT, exceptions, leave/holiday overlay).
- Triggered after clock events for the day settle (end_offset) and nightly for the prior day; manual corrections re-trigger it.
- Exception dashboard query reads `daily_attendance WHERE is_exception` scoped by manager site.

**Testing**:
- `Integration (testcontainers): insert events, refresh aggregate, run reconcile → daily_attendance row matches compute_day`.
- `Integration: approved leave on a date → status=leave, no absent exception`.
- `Integration: holiday on date → status=holiday`.
- `Unit: manager exception query is site-scoped (RLS + scope)`.

---

## Phase 5: Payroll Export & HRMS Integration

### Purpose
Make attendance actionable downstream: export verified, computed time to payroll via CSV and live connectors (Xero, QuickBooks for MVP). After this phase HR can run a pay period and push hours to payroll.

### Tasks

#### 5.1 — Export engine and CSV exporter

**What**: Compute a pay-period dataset and emit a configurable CSV.

**Design**:
- `payroll_exports` table from Suggestion 4 (`export_config` JSONB: field mappings, filters, format).
- `payroll.build_period(org, period_start, period_end, filters) -> list[PayrollRow]` aggregates `daily_attendance` into regular/OT/leave hours per employee with `payroll_id` mapping.
- `POST /api/v1/payroll/exports` creates a record + enqueues a Celery job; `GET /api/v1/payroll/exports/{id}` returns status + `file_url`.
- CSV columns configurable; timestamps ISO 8601; numeric hours to 2dp.

**Testing**:
- `Unit: build_period aggregates regular vs OT hours correctly for multi-day period`.
- `Unit: CSV matches configured field mapping and header order`.
- `Integration: create export → job completes, status=completed, file retrievable`.
- `Unit: employee without payroll_id mapping → row flagged, export warns not crashes`.

#### 5.2 — Connector framework + Xero & QuickBooks

**What**: Pluggable payroll-connector interface with OAuth2 connectors for Xero and QuickBooks.

**Design**:
```python
class PayrollConnector(Protocol):
    name: str
    async def authorize_url(self, org_id: UUID) -> str: ...
    async def exchange_code(self, org_id: UUID, code: str) -> None: ...
    async def push_timesheets(self, rows: list[PayrollRow], period: Period) -> PushResult: ...
```
- Xero: OAuth2 PKCE, Payroll API timesheets (AU/NZ/UK variants by org region). QuickBooks: OAuth2, TimeActivities.
- Tokens stored encrypted (Vault); refresh handled transparently. Rate limits respected (Workday-style 10 rps cap generalised via a token-bucket per connector).
- Failures retried with exponential backoff (Celery); `PushResult{pushed, failed[], errors}` surfaced to the UI.

**Testing**:
- `Integration (mocked Xero API): push 10 rows → 200, PushResult.pushed=10`.
- `Integration (mocked): 429 rate limit → backoff + retry succeeds`.
- `Unit: OAuth token refresh on 401 then retry`.
- `Unit: connector registry resolves 'xero' → XeroConnector`.

---

## Phase 6: Web Console — Admin, Manager & Self-Service

### Purpose
Deliver the SaaS-polished UI incumbents lack: HR admin console, manager exception dashboard, and employee self-service portal with WebAuthn login. After this phase non-technical staff operate the system without touching the API.

### Tasks

#### 6.1 — App shell, generated API client, WebAuthn login

**What**: Next.js app with auth, role-aware navigation, and a typed client generated from the OpenAPI spec.

**Design**:
- Generate `web/lib/api` from `/openapi.json` (orval or openapi-typescript). TanStack Query for data.
- Login: OIDC SSO for staff; WebAuthn passkey registration/assertion for employees (`/auth/webauthn/*` endpoints added in backend; uses `py_webauthn`). Aligns NIST 800-63-4 AAL2.
- Role-gated routes: employee → self-service only; manager → site dashboards; hr_admin → org console; sys_admin → settings/devices/integrations.

**Testing**:
- `E2E (Playwright): login as hr_admin → admin nav visible; login as employee → only self-service`.
- `E2E: WebAuthn registration + assertion (virtual authenticator) → authenticated session`.
- `Unit: generated client types compile against the committed openapi snapshot`.

#### 6.2 — Manager exception dashboard

**What**: Real-time view of late/absent/early-departure/anomaly exceptions for a manager's site, with approve/correct actions.

**Design**:
- Reads `GET /api/v1/attendance/exceptions?site_id&date_range`. Cards per exception; actions: approve, request correction, add note.
- Correction flow triggers `reconcile_day` and writes audit. Live updates via polling (SSE/WebSocket optional later).

**Testing**:
- `E2E: seed late arrival → appears on dashboard; approve → disappears, audit row written`.
- `E2E: manager sees only their site's exceptions`.
- `Unit: correction submits to API and invalidates the query cache`.

#### 6.3 — Employee self-service portal

**What**: Employee views own attendance/leave history, requests leave, submits clock corrections, manages consent.

**Design**:
- Routes: history (calendar of `daily_attendance`), leave request form, correction request, consent centre (view granted consents, revoke → triggers opt-out flow from 2.1).
- All reads RLS-scoped to the employee; corrections create pending items for manager approval.

**Testing**:
- `E2E: employee requests leave → appears as pending for manager`.
- `E2E: employee revokes facial consent → confirmation shown, template deactivation enqueued`.
- `E2E: employee cannot see another employee's history (403/empty)`.

---

## Phase 7: Mobile App — Capture, On-Device Match & Offline Queue

### Purpose
Deliver the mobile-first MVP clock-in experience with on-device biometrics and an offline-first queue. After this phase a field worker can clock in via face/fingerprint with GPS, fully offline, syncing on reconnect — closing the MVP.

### Tasks

#### 7.1 — Capture, liveness & on-device matching

**What**: Capture face/fingerprint on device, run liveness, match against the locally cached template, and never transmit raw images.

**Design**:
- `expo-local-authentication` for device Face ID / fingerprint as the primary verification (device attests identity); `expo-camera` selfie path runs passive liveness (blink/parallax) for kiosk-style use.
- On-device template + salt cached encrypted (expo-secure-store); matching produces `match_score` + `liveness_passed`; only the transformed template id, scores, and metadata go to the server.
- Kiosk mode: shared tablet, employee selects/enters id then verifies.

**Testing**:
- `Unit (Jest): match function returns score; below threshold → blocked locally`.
- `Detox E2E: enrol then clock-in on device → event queued with match_score, no image in payload`.
- `Unit: liveness blink challenge fails on static photo fixture`.

#### 7.2 — Offline queue & sync engine

**What**: Local SQLite queue of clock events that syncs idempotently to `/sync/events`.

**Design**:
- WatermelonDB/SQLite stores pending events with a client-generated `idempotency_key` and `event_time`.
- Sync engine: on connectivity, batch-POST to `/api/v1/sync/events`, then clear/mark each event from the per-event status response (Phase 3.3 contract). Exponential backoff; conflict-free because of idempotency keys.
- GPS captured at event time even when offline.

**Testing**:
- `Detox E2E: airplane mode → 3 clock events queued locally; reconnect → all sync, queue empties`.
- `Unit: duplicate sync response marks event done without re-sending`.
- `Unit: backoff increases on repeated network failure`.

---

## Phase 8: Hardware Terminal Adapters (v1.1)

### Purpose
Break vendor lock-in: a unified device-integration API with adapters for ZKTeco, Suprema, and eSSL terminals, plus device health monitoring. After this phase the platform ingests events from physical terminals under the same pipeline as mobile.

### Tasks

#### 8.1 — Device adapter framework & registry

**What**: Common adapter interface translating vendor protocols/payloads into the internal clock-event model.

**Design**:
```python
class DeviceAdapter(Protocol):
    device_type: str
    def parse_event(self, raw: dict) -> ClockEventIn: ...
    async def pull_logs(self, device: Device, since: datetime) -> list[ClockEventIn]: ...
    async def push_user(self, device: Device, employee: Employee, template: bytes) -> None: ...
    def verify_signature(self, request: Request) -> bool: ...
```
- `devices.capabilities` JSONB (Suggestion 3) holds vendor model/firmware/protocol.
- Inbound: `POST /api/v1/webhooks/devices/{device_id}` (push protocol) validates signature, parses via the registered adapter, runs the Phase 3.1 pipeline. Outbound pull on a schedule for SDK/SQL-protocol devices.

**Testing**:
- `Unit (fixture): ZKTeco sample push payload → correct ClockEventIn`.
- `Unit: Suprema BioStar X JSON event → correct ClockEventIn`.
- `Unit: invalid device signature → 401, no event recorded`.
- `Integration: pull_logs since timestamp → only newer events ingested, idempotent`.

#### 8.2 — Device heartbeats & health monitoring

**What**: Ingest heartbeats, surface device health and offline detection.

**Design**:
- `device_heartbeats` + `device_health_hourly` continuous aggregate from Suggestion 4. `POST /api/v1/devices/{id}/heartbeat`.
- A worker marks a device `offline` when `last_heartbeat` exceeds a threshold and raises an alert (Keka-style graceful failure handling). `pending_sync_count` surfaced on the device-health dashboard.

**Testing**:
- `Integration: heartbeat → device.last_heartbeat updated, health aggregate populated`.
- `Unit: no heartbeat beyond threshold → status flips to offline + alert emitted`.

---

## Phase 9: AI Anomaly Detection (v1.1)

### Purpose
Deliver the headline differentiator: ML-based detection of buddy-punch clusters, overtime spikes, and schedule drift, with LLM-generated explanations and a manager review workflow. After this phase the system surfaces fraud patterns that static rules miss.

### Tasks

#### 9.1 — Detector framework & batch scoring

**What**: Pluggable detectors writing scored anomalies to the `anomaly_scores` hypertable.

**Design**:
- `anomaly_scores` hypertable from Suggestion 4 (`details` JSONB, severity, model_version, review fields).
```python
class AnomalyDetector(Protocol):
    detection_type: str
    def score(self, ctx: DetectionContext) -> list[AnomalyScore]: ...
```
- Detectors:
  - **Buddy-punch cluster**: DBSCAN over (employee, device, time) — flags pairs repeatedly clocking within seconds at the same device.
  - **Overtime spike**: IsolationForest on per-employee weekly OT vs personal/department baseline.
  - **Schedule drift**: linear-trend (statsmodels) on clock-in times creeping later while staying within policy.
- Nightly Celery batch (`run_detectors`) over a rolling window; results upserted with `status=open`.

**Testing**:
- `Unit (synthetic fixture): two employees clocking within 20s, 5 days → buddy-punch flagged with implicated ids`.
- `Unit: injected OT spike → IsolationForest flags it; normal variance → not flagged`.
- `Unit: clock-in drifting +2 min/week over 6 weeks → schedule_drift flagged with slope`.
- `Integration: run_detectors writes anomaly_scores rows`.

#### 9.2 — Anomaly review workflow & LLM explanations

**What**: Manager-facing feed with confirm/dismiss and plain-language explanations.

**Design**:
- `GET /api/v1/anomalies?status=open&severity` (site-scoped); `POST /api/v1/anomalies/{key}/confirm|dismiss` with notes → audit.
- `LLMClient.explain(anomaly_details) -> str` produces a human explanation from the structured `details`; cached on the record. System prompt constrains output to the provided facts (no fabrication).
- Confirmed/dismissed outcomes are logged as labels to bootstrap future supervised models (and to retrain liveness from confirmed spoof attempts).

**Testing**:
- `Integration (mocked LLM): explain → deterministic string from fixture details; no PII beyond provided fields`.
- `Unit: confirm/dismiss updates status + writes audit + stores label`.
- `Unit: anomaly feed is site-scoped for managers`.

---

## Phase 10: Multi-Jurisdiction Compliance Automation (v1.1)

### Purpose
Automate the compliance burden incumbents handle manually: per-jurisdiction consent templates, automated retention/deletion via Timescale retention policies, DSAR/right-to-erasure, and crypto-shredding. After this phase the system enforces BIPA/GDPR/PDPA/POPIA obligations without manual sweeps.

### Tasks

#### 10.1 — Jurisdiction rule engine & consent templates

**What**: Configurable per-jurisdiction rules driving consent required-fields and retention defaults.

**Design**:
- `JurisdictionRules` registry (GDPR, BIPA, PDPA, POPIA): required consent fields, default retention, deletion grace, legal basis vocabulary. Drives the 2.1 consent-template endpoint and validation.
- Versioned consent text per jurisdiction stored with hashes; `consent_text_hash` validated on capture.

**Testing**:
- `Unit: GDPR rules require legal_basis + purpose_limitation; BIPA require written_release + destruction_schedule`.
- `Unit: unknown jurisdiction → falls back to default with warning`.

#### 10.2 — Automated retention, deletion & DSAR

**What**: Scheduled enforcement of deletion-due templates, right-to-erasure, and data-subject access exports.

**Design**:
- Daily worker queries `biometric_consents WHERE deletion_due_at <= now() AND revoked_at IS NULL` plus revoked consents past grace → deactivate + delete templates, crypto-shred keys (destroy Vault key version → ciphertext unrecoverable, Suggestion 2's crypto-shredding idea applied to template blobs).
- Timescale retention policies on `clock_events`/`audit_log`/`anomaly_scores` per Suggestion 4, configurable per org/jurisdiction.
- `POST /api/v1/compliance/dsar/{employee_id}` → packaged export of the subject's data; `POST /api/v1/compliance/erase/{employee_id}` → erase biometric data while retaining the audit record of the erasure itself.
- 72-hour breach-notification workflow stub (GDPR Art. 33): an incident record + notification checklist.

**Testing**:
- `Integration: template past deletion_due_at → deleted, key version destroyed, audit row of deletion retained`.
- `Integration: erase request → biometric templates gone, audit log of erasure preserved`.
- `Integration: DSAR export contains the subject's records and excludes other subjects`.
- `Unit: retention policy interval derived from org jurisdiction config`.

---

## Phase 11: Leave/PTO, NL Reporting & MCP Server (v1.1 → backlog)

### Purpose
Round out the platform with the leave module, conversational analytics, and an MCP server so any AI assistant can query attendance securely. After this phase HR can ask questions in plain English and integrate the platform with AI tooling.

### Tasks

#### 11.1 — Leave & PTO module

**What**: Leave types, accrual, balances, request/approval workflow integrated with attendance.

**Design**:
- `leave_types`, `leave_requests`, `leave_balances` from Suggestion 4 (`policy`/`details`/`balance` JSONB). Accrual worker updates balances monthly; approved leave overlays `daily_attendance` (status=leave) consumed by reconciliation (4.3) and payroll (5.1).
- Endpoints: CRUD leave types; `POST /api/v1/leave/requests`; manager approve/reject; balance query.

**Testing**:
- `Unit: accrual adds accrual_rate per month, capped at max; carry-forward expiry applied`.
- `Unit: request exceeding balance → 422`.
- `Integration: approved leave → daily_attendance status=leave, excluded from absence exceptions`.

#### 11.2 — Natural-language reporting

**What**: Plain-English questions over attendance data, answered via guarded SQL generation.

**Design**:
- `POST /api/v1/reporting/ask` `{question}` → LLM maps to a parameterised, read-only query from an allowlisted template set (e.g. absence-by-department, OT-by-site, drift-by-employee). Never free-form SQL; the LLM only fills parameters and selects a template. Results returned + the chosen template id for transparency.
- Tenant/RLS scoping enforced regardless of generated query.

**Testing**:
- `Integration (mocked LLM): "departments with rising absenteeism this quarter" → maps to absence-trend template, returns rows`.
- `Security: attempted prompt-injection to read another org's data → blocked by RLS + allowlist`.
- `Unit: unmappable question → graceful "cannot answer" with suggestions`.

#### 11.3 — Read-only MCP server

**What**: An MCP server exposing attendance summaries, anomaly alerts, and shift data as tools.

**Design**:
- `bas.mcp` server exposing read-only tools (`get_attendance_summary`, `list_open_anomalies`, `get_employee_schedule`) backed by the reporting API, authenticated by a scoped service token. No raw biometric data or template access exposed (OWASP excessive data exposure).

**Testing**:
- `Integration: MCP tool call returns summary JSON; biometric template tool does not exist`.
- `Unit: MCP service token scoped read-only; write attempt → rejected`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Auth            ─── required by everything
    │
Phase 2: Enrolment, Consent & Template Security ─── requires Phase 1
    │
Phase 3: Clock Events, Geofencing & Sync       ─── requires Phase 2
    │
Phase 4: Scheduling, Work Rules & Attendance   ─── requires Phase 3
    │
    ├── Phase 5: Payroll & HRMS Integration     ─── requires Phase 4
    ├── Phase 6: Web Console                     ─── requires Phase 4 (parallel with 5, 7)
    └── Phase 7: Mobile App                      ─── requires Phase 3 (parallel with 5, 6)  ← completes MVP
         │
         ├── Phase 8: Hardware Terminal Adapters ─── requires Phase 3 (parallel with 9, 10)
         ├── Phase 9: AI Anomaly Detection       ─── requires Phase 4 (parallel with 8, 10)
         ├── Phase 10: Compliance Automation      ─── requires Phase 2 (parallel with 8, 9)
         └── Phase 11: Leave, NL Reporting, MCP   ─── requires Phase 4 (and 9 for anomaly MCP tool)
```

**MVP = Phases 1–7.** Phases 8–11 are v1.1 and beyond.

**Parallelism opportunities:**
- After Phase 4: Phases 5, 6, and 7 can be built concurrently (separate backend/web/mobile workstreams).
- After the MVP: Phases 8, 9, and 10 are independent and can be built concurrently.
- Phase 11.1 (leave) can start any time after Phase 4; 11.2/11.3 after Phase 9.

---

## Definition of Done (per phase)

1. All tasks in the phase implemented.
2. All unit and integration tests pass (`pytest`, `vitest`, `jest`/`detox` as applicable).
3. Linting and formatting pass (`ruff`, `mypy --strict`, `bandit`; `eslint`/`prettier`/`tsc`).
4. `docker compose up` builds and the affected services start healthy.
5. The phase's feature works end-to-end (demonstrated by an E2E or integration test).
6. New config options documented in `.env.example` and README.
7. New/changed API endpoints appear in the regenerated OpenAPI 3.1 spec, and a spec snapshot is committed under `openapi/`.
8. Alembic migrations created and reversible; hypertables/continuous-aggregates/retention policies verified via `timescaledb_information`.
9. Every mutating operation writes an immutable `audit_log` entry.
10. No endpoint exposes raw biometric images or template blobs (OWASP API Security Top 10 check passes).
```
