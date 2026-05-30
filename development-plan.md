# Media Rights Management — Phased Development Plan

> Project: 368-media-rights-management · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the three data-model suggestions. It adopts **Data Model Suggestion 1 (Entity-Centric Normalized Relational)** as the canonical production schema because the two dominant access patterns — multi-dimensional avails queries and royalty waterfall calculation — are best served by indexed relational joins with referential integrity across the rights chain (`title → grant → rule → calculation → statement`). Audit-first concepts from Suggestion 3 (immutable `audit_log`, event-style provenance for rights changes) are layered on top rather than replacing the relational core. The JSONB-hybrid ideas from Suggestion 2 are retained selectively for `clause_extraction_json`, `odrl_policy_json`, and `escalation_json` where the data is genuinely schema-flexible.

---

## Core Requirements (Synthesis)

**What it does.** An AI-native, vertical-agnostic platform for content licensing, rights tracking, and royalty calculation across music, film, publishing, games, and sports. It tracks rights grants across territory × platform × format × language × time-window dimensions, detects conflicts and holdbacks, computes royalties from reported sales data, and generates payee statements.

**Who uses it.** Rights managers (catalogue + avails), royalty accountants (calculation + statements), licensees (avails inquiry, portal submission), payees (statement receipt), and auditors (read-only audit trail). Primary buyers are mid-market publishers, music labels/publishers, indie film distributors, sports/brand licensors, and cross-vertical content groups.

**Key differentiators.** NLP clause extraction from PDF contracts; real-time avails API consumable by sales/CRM; configurable royalty rule DSL; vertical-agnostic data model; anomaly detection on incoming sales statements; natural-language avails queries; DDEX/ONIX/EIDR/CWR interoperability; optional cryptographic audit trail; MCP server exposing rights tools to LLM agents; `ai_training` modelled as a first-class right type.

**MVP scope (must-have).** Rights catalogue with full dimensions + conflict detection; contract storage with structured metadata; royalty engine (flat, %, escalators, splits, advances, recoupment, MGs); statement generation (PDF + CSV) with multi-currency + historical FX; sales importers for ≥3 DSP/retail formats; RBAC (owner/licensee/payee/auditor) with full audit trail; REST API for avails + rights CRUD.

**v1.1 (should-have).** AI clause extraction; self-service licensee/payee portal with e-signature; anomaly detection; ONIX/DDEX/EIDR import-export; configurable royalty DSL; renewal/obligation calendar with alerts.

**Backlog (nice-to-have).** Cryptographic audit trail; smart-contract disbursement; NL avails search; AI-training-data licensing module; multi-language translation; predictive renewals; MCP server.

**Deployment model.** Cloud-native multi-tenant SaaS with a self-host (Docker Compose) option. REST API first; SSO via OIDC; ERP/finance integration surface.

**Standards to implement.** ISO 3901/15707/15706/21047/26324/27729 (identifiers), ISO 3166/4217/639 (codes), W3C ODRL 2.2 (canonical rights representation), OpenAPI 3.1, JSON Schema 2020-12, OAuth 2.0/OIDC + JWT (RFC 7519/6749), DDEX ERN/DSR, ONIX 3.0, EIDR, CWR, OWASP API Security Top 10 (2023), SOC 2 audit-trail expectations, GDPR, eIDAS/ESIGN for e-signature.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | Python 3.12 | Domain is NLP/LLM-heavy (clause extraction, anomaly detection, NL avails). Python has the richest ecosystem for LLM SDKs, PDF parsing, and data ingestion, which are the project's differentiators. |
| API framework | FastAPI | Generates OpenAPI 3.1 natively (a stated standard), async-first for high-volume DSP webhooks/ingestion, Pydantic validation maps directly to JSON Schema 2020-12 for rights/royalty payloads. |
| Data validation | Pydantic v2 | Single source of truth for request/response models, config, and the royalty DSL AST. Emits JSON Schema for the OpenAPI contract. |
| Database | PostgreSQL 16 | The rights model is inherently relational and multi-dimensional. Needs `TEXT[]` arrays + GIN indexes for territory/platform avails queries, range partitioning for `sales_data` and `audit_log`, `JSONB` for ODRL/escalation/clause data, and full-text GIN for title search. SQLite cannot do array overlap or partitioning. |
| ORM / migrations | SQLAlchemy 2.0 + Alembic | Mature async support, explicit migrations required for the 13-table schema; Alembic versioned migrations satisfy the "DB migrations created" DoD item. |
| Task queue | Celery + Redis | DSP ingestion (millions of rows), AI clause extraction, anomaly detection, statement PDF generation, and FX fetching are long-running/async. Redis doubles as cache and rate-limit store. |
| Object storage | S3-compatible (MinIO self-host / AWS S3 SaaS) via `boto3` | Contract PDFs and generated statement PDFs must live outside the DB; `document_url` columns reference object keys. MinIO keeps self-host single-stack. |
| LLM provider | Anthropic Claude via official SDK, behind an `LLMClient` abstraction | Clause extraction, classification, anomaly explanation, and NL avails. Abstraction allows swapping providers per-tenant and enforces per-tenant isolation (no cross-tenant fine-tuning) per the IP notes in features.md. |
| PDF parsing | `pypdf` + `pdfplumber` | Extract text/layout from heterogeneous contract PDFs before LLM clause extraction. |
| PDF generation | WeasyPrint (HTML/CSS → PDF) | Statement rendering from Jinja2 templates; clean multi-currency/multi-language layout. |
| FX rates | `httpx` client against ECB / exchangerate.host, cached in `fx_rates` table | Historical exchange-rate conversion (ISO 4217) for statements; rates stored per-day for reproducible recalculation. |
| Frontend | Next.js 15 (React, TypeScript) + shadcn/ui + TanStack Query | Dashboard for rights managers/accountants, plus the licensee/payee self-service portal. Consumes the REST API. Server components for catalogue grids, client components for avails filters. (MVP can ship API-only; UI lands in Phase 9.) |
| Auth | OAuth 2.0 / OIDC (Authlib) + JWT (RFC 7519), API keys for machine clients | SSO for enterprise rights-holders; delegated portal access; FAPI-leaning posture for royalty payment flows. |
| E-signature | DocuSign / Adobe Sign via adapter interface | eIDAS/ESIGN-compliant contract execution in the licensing workflow. |
| Containerisation | Docker + docker-compose | Self-host story (api, worker, postgres, redis, minio, web); reproducible CI. |
| Testing | pytest + pytest-asyncio + testcontainers + httpx.AsyncClient + Schemathesis | Unit + integration against ephemeral Postgres/Redis containers; Schemathesis fuzzes the OpenAPI contract; Playwright for portal E2E. |
| Code quality | Ruff (lint + format), mypy (strict), `bandit` (security) | OWASP/ASVS posture; type safety across the royalty engine where money math matters. |
| Package manager | `uv` | Fast, reproducible locking for the Python backend; `pnpm` for the web app. |
| Money representation | Integer cents (`BIGINT`) + `Decimal` for rates | Never use floats for money. Rates stored as `NUMERIC(7,4)`; all monetary fields are `*_cents BIGINT`. |
| MCP server | Python MCP SDK | Exposes `rights_avails`, `royalty_calc`, and `contracts` tools/resources to LLM clients (backlog phase). |

### Project Structure

```
media-rights-management/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── alembic.ini
├── .env.example
├── README.md
├── migrations/                      # Alembic versioned migrations
│   ├── env.py
│   └── versions/
├── src/
│   └── mrm/
│       ├── __init__.py
│       ├── main.py                  # FastAPI app factory, router registration
│       ├── config.py                # Pydantic Settings
│       ├── db.py                    # async engine, session, base
│       ├── deps.py                  # FastAPI dependencies (db, current_user, tenant)
│       ├── security/
│       │   ├── auth.py              # OIDC, JWT, API-key verification
│       │   ├── rbac.py             # role → permission matrix, scoping
│       │   └── passwords.py
│       ├── models/                  # SQLAlchemy ORM (one module per aggregate)
│       │   ├── org.py  user.py  party.py  title.py  contract.py
│       │   ├── rights.py  royalty.py  sales.py  ai.py  audit.py  fx.py
│       ├── schemas/                 # Pydantic request/response models
│       ├── repositories/            # data-access layer (queries, avails, conflicts)
│       ├── services/                # business logic
│       │   ├── avails.py            # avails search + conflict detection
│       │   ├── royalty/             # engine package
│       │   │   ├── engine.py        # waterfall orchestration
│       │   │   ├── dsl.py           # royalty rule DSL parser/evaluator
│       │   │   ├── rules.py         # rule-type handlers
│       │   │   └── recoupment.py
│       │   ├── statements.py        # statement assembly
│       │   ├── ingestion/           # sales importers
│       │   │   ├── base.py  spotify.py  apple.py  ddex_dsr.py  csv_generic.py
│       │   │   ├── fx.py
│       │   │   └── anomaly.py
│       │   ├── contracts.py
│       │   ├── obligations.py       # renewal/obligation calendar
│       │   ├── interop/             # DDEX / ONIX / EIDR / CWR
│       │   │   ├── ddex_ern.py  onix.py  eidr.py  cwr.py  odrl.py
│       │   └── ai/                  # LLM features
│       │       ├── client.py        # LLMClient abstraction
│       │       ├── clause_extraction.py
│       │       ├── classification.py
│       │       └── nl_avails.py
│       ├── api/                     # FastAPI routers
│       │   └── v1/
│       │       ├── titles.py  contracts.py  rights.py  avails.py
│       │       ├── royalties.py  statements.py  sales.py  parties.py
│       │       ├── auth.py  portal.py  webhooks.py  interop.py
│       ├── workers/                 # Celery tasks
│       │   ├── celery_app.py  ingestion_tasks.py  ai_tasks.py
│       │   └── statement_tasks.py  obligation_tasks.py
│       ├── rendering/               # statement templates
│       │   └── templates/statement.html.j2
│       ├── mcp/                     # MCP server (backlog)
│       │   └── server.py
│       └── utils/                   # money, dates, territory expansion, ids
├── web/                             # Next.js app (Phase 9)
│   ├── package.json
│   └── src/app/...
└── tests/
    ├── conftest.py
    ├── fixtures/                    # sample PDFs, DSP reports, ONIX/DDEX files
    ├── unit/  integration/  e2e/
```

---

## Phase 1: Foundation & Tenancy

### Purpose
Establish the runnable skeleton: app factory, configuration, async DB connectivity, migrations, multi-tenancy (`org_id` scoping), the audit-log substrate, and CI. After this phase the service boots, connects to Postgres/Redis, exposes a health check and OpenAPI docs, and has the organisations/users tables migrated — every later entity hangs off `organisations`.

### Tasks

#### 1.1 — Project scaffold, config, app factory

**What**: A bootable FastAPI app with typed settings and a `/healthz` endpoint.

**Design**:
```python
# config.py
class Settings(BaseSettings):
    database_url: str                     # postgresql+asyncpg://...
    redis_url: str = "redis://localhost:6379/0"
    s3_endpoint: str | None = None
    s3_bucket: str = "mrm"
    jwt_secret: str
    jwt_algorithm: str = "HS256"
    access_token_ttl_seconds: int = 3600
    oidc_issuer: str | None = None
    llm_api_key: str | None = None
    llm_model: str = "claude-opus-4"
    default_currency: str = "USD"         # ISO 4217
    environment: Literal["dev","test","staging","prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="MRM_", env_file=".env")
```
`create_app() -> FastAPI` registers routers, exception handlers (uniform `{error: {code, message, details}}` body), CORS, and request-ID middleware. `GET /healthz` returns `{status, db, redis}` after pinging both.

**Testing**:
- `Unit: Settings loads from env with MRM_ prefix → correct typed values, defaults applied`
- `Unit: missing required jwt_secret → ValidationError naming the field`
- `Integration: GET /healthz with DB+Redis up → 200, {status:"ok", db:"ok", redis:"ok"}`
- `Integration: GET /healthz with DB down → 503, db:"error"`
- `Integration: GET /openapi.json → 200, openapi field == "3.1.0"`

#### 1.2 — Async DB layer + Alembic

**What**: SQLAlchemy 2.0 async engine/session, declarative `Base`, and Alembic wired for autogenerate.

**Design**:
```python
# db.py
engine = create_async_engine(settings.database_url, pool_size=10, pool_pre_ping=True)
SessionLocal = async_sessionmaker(engine, expire_on_commit=False)
class Base(DeclarativeBase):
    metadata = MetaData(naming_convention={...})  # stable constraint names for migrations
async def get_session() -> AsyncIterator[AsyncSession]: ...
```
A `TimestampMixin` (`created_at`, `updated_at`) and `OrgScopedMixin` (`org_id: Mapped[UUID]`) are shared by entities. UUID PKs default `gen_random_uuid()` (enable `pgcrypto`).

**Testing**:
- `Integration (testcontainers): alembic upgrade head on empty DB → all tables present`
- `Integration: alembic downgrade base then upgrade head → idempotent, no diff via autogenerate`
- `Unit: get_session yields a session and closes it on exit (no leaked connections)`

#### 1.3 — Organisations & Users models + migration

**What**: `organisations` and `users` tables (from Data Model 1) with the first migration.

**Design**: Implement `organisations` and `users` exactly as specified in `data-model-suggestion-1.md` (org_type/role CHECKs, `idx_users_org`). Add `users.password_hash TEXT` and `users.external_subject TEXT` (OIDC `sub`) for auth.

**Testing**:
- `Integration: insert org + user → row retrievable; FK users.org_id enforced`
- `Integration: duplicate user email → IntegrityError (unique)`
- `Integration: invalid role value → CHECK violation`

#### 1.4 — Audit-log substrate (partitioned)

**What**: The `audit_log` table (range-partitioned by `created_at`) and a writer service used by all later mutations.

**Design**: Implement `audit_log` per Data Model 1. Provide `AuditWriter.record(session, *, actor_type, actor_id, action, entity_type, entity_id, changes, ip)`. Create monthly partitions via a `create_audit_partition(year, month)` helper; migration seeds the current + next 3 months. Mutations call the writer in the same transaction so audit and data commit atomically (SOC 2 expectation).

**Testing**:
- `Unit: AuditWriter builds correct row with changes diff dict`
- `Integration: write to current-month partition → row lands in correct partition (check via pg_class)`
- `Integration: write with timestamp outside any partition → caught, partition auto-created, retried`

#### 1.5 — CI, lint, type-check, Docker

**What**: Reproducible CI plus Docker Compose stack.

**Design**: `docker-compose.yml` services: `api`, `worker`, `postgres:16`, `redis:7`, `minio`. CI runs `ruff check`, `ruff format --check`, `mypy --strict src`, `bandit -r src`, `pytest` (with testcontainers). Multi-stage `Dockerfile` using `uv`.

**Testing**:
- `CI: ruff/mypy/bandit all green on the skeleton`
- `Integration: docker compose up → api /healthz reachable; build succeeds`

---

## Phase 2: Catalogue & Parties

### Purpose
Model the things being licensed (titles, contributors) and the parties to deals (rights holders, licensees, payees) with their industry identifiers. This is the noun layer everything else references. After this phase the catalogue is fully CRUD-able via the API with identifier-based lookup and full-text title search.

### Tasks

#### 2.1 — Parties model + CRUD API

**What**: `parties` table and `/v1/parties` endpoints.

**Design**: Implement `parties` per Data Model 1 (party_type CHECK, `isni`, `ipi_number`, `payment_method`, indexed ISNI/IPI). Endpoints:
- `POST /v1/parties` body `PartyCreate{party_type, name, isni?, ipi_number?, tax_id?, country?, currency_preferred?, payment_method?, payment_details?}` → `201 PartyOut`
- `GET /v1/parties?type=&q=&isni=&page=&size=` → paginated `PartyOut`
- `GET/PATCH/DELETE /v1/parties/{id}` (DELETE is soft → `is_active=false`)

All queries are `org_id`-scoped from the auth context. ISNI validated against the ISO 27729 16-digit checksum.

**Testing**:
- `Unit: ISNI checksum validator accepts valid, rejects bad check digit`
- `Integration: POST valid party → 201, persisted, audit_log row written`
- `Integration: GET ?isni= → returns matching party only within tenant`
- `Integration: party from another org not returned (tenant isolation)`
- `Integration: DELETE → is_active=false, still retrievable with include_inactive=true`

#### 2.2 — Titles model + CRUD + identifier validation

**What**: `titles` table and `/v1/titles` endpoints with industry-identifier validation.

**Design**: Implement `titles` per Data Model 1 (all identifier columns + partial indexes + `to_tsvector` GIN + `parent_title_id`). Validators:
- ISRC (ISO 3901) format `CCXXXNNNNNNN`; ISWC (ISO 15707) `T-NNNNNNNNN-C`; ISAN (ISO 15706); ISBN-13 checksum; EIDR DOI shape.
- `vertical`/`title_type` consistency check (e.g. `music_recording` ⇒ `vertical=music`).

Endpoints mirror parties plus `GET /v1/titles/{id}/children` (uses `parent_title_id`).

**Testing**:
- `Unit: ISRC/ISWC/ISBN-13 validators (valid + malformed + bad checksum)`
- `Unit: vertical/title_type mismatch → 422 with field detail`
- `Integration: full-text search GET ?q=midnight → ranked matches`
- `Integration: create episode with parent series → children endpoint returns it`
- `Integration: lookup by isrc/isbn returns exactly one title`

#### 2.3 — Title contributors + ownership splits

**What**: `title_contributors` join table and nested endpoints.

**Design**: Implement `title_contributors` per Data Model 1 (`contributor_role` CHECK, `ownership_pct NUMERIC(7,4)`, `territory_scope TEXT[]`, `is_controlled`).
- `POST /v1/titles/{id}/contributors`, `GET`, `PATCH`, `DELETE`.
- Service validates that controlled splits per role sum to ≤ 100.0000 (warn, do not block — uncontrolled shares are legitimate).

**Testing**:
- `Unit: split-sum check returns warning when controlled shares > 100%`
- `Integration: add 2 composers at 50/50 → both persist, sum ok`
- `Integration: delete contributor → audit row, split warning recomputed`

---

## Phase 3: Contracts & Rights Grants

### Purpose
The heart of the platform. Model contracts and the multi-dimensional rights grants they create, with the ODRL canonical representation. After this phase the system holds structured deal data and the grants needed for avails and royalties — the core IP of the product.

### Tasks

#### 3.1 — Contracts model + lifecycle + CRUD

**What**: `contracts` table, status state machine, and `/v1/contracts` endpoints.

**Design**: Implement `contracts` per Data Model 1 (contract_type/status CHECKs, advance/MG cents, `clause_extraction_json`, `obligations_json`, `signature_json`, unique `(org_id, contract_ref)`). State machine:
```
draft → negotiation → executed → active → {expired | terminated | renewed}
```
`POST /v1/contracts/{id}:transition` body `{to_status}` validated against allowed edges; illegal transition → `409`. Each transition writes an `audit_log` entry with before/after. PDF upload: `POST /v1/contracts/{id}/document` (multipart) → stores to S3, sets `document_url`.

**Testing**:
- `Unit: transition matrix — legal edge ok, illegal edge raises`
- `Integration: create draft → executed → active happy path`
- `Integration: active → draft rejected with 409`
- `Integration: duplicate contract_ref in same org → 409`
- `Integration: PDF upload → object stored, document_url set, audit row`

#### 3.2 — Rights grants model + CRUD

**What**: `rights_grants` table and nested `/v1/contracts/{id}/grants` + `/v1/grants/{id}`.

**Design**: Implement `rights_grants` per Data Model 1 (right_type CHECK incl. `ai_training`, `territories/excluded_territories/platforms/formats/languages TEXT[]`, `window_start/end`, exclusivity, holdback, `odrl_policy_json`, the `idx_grants_avails`/GIN territory indexes). Territory expansion utility maps macro codes (`WW`, `EU`, `DACH`, `NORDICS`) → ISO 3166 alpha-2 sets used during storage-time normalisation and query-time matching.

```python
def expand_territories(codes: list[str]) -> set[str]:
    # "WW" → all ISO-3166-1 a2; "DACH" → {DE,AT,CH}; "EU" → 27 members; passthrough a2
```

**Testing**:
- `Unit: expand_territories WW/EU/DACH/passthrough/unknown-code→error`
- `Integration: create grant with territories=[DACH] → stored, GIN-searchable as DE`
- `Integration: window_end < window_start → 422`
- `Integration: grant references title in another org → 422 (cross-tenant FK guard)`

#### 3.3 — ODRL serialisation

**What**: Map a `rights_grant` to/from a W3C ODRL 2.2 policy JSON.

**Design**: `services/interop/odrl.py`:
```python
def grant_to_odrl(grant: RightsGrant) -> dict   # ODRL Policy with permission + constraints
def odrl_to_grant_fields(policy: dict) -> dict   # inverse for import
```
Right types map to ODRL actions; territories/window/exclusivity map to ODRL `constraint`s (`spatial`, `dateTime`, `count`); `ai_training` maps to a custom action IRI under the project vocabulary. Result stored in `rights_grants.odrl_policy_json` on write. `GET /v1/grants/{id}/odrl` returns the policy.

**Testing**:
- `Unit: grant_to_odrl → valid ODRL with permission.target=title IRI, spatial constraint per territory`
- `Unit: round-trip grant → odrl → grant_fields preserves dimensions`
- `Fixture: emitted ODRL validates against ODRL JSON-LD context`

---

## Phase 4: Avails & Conflict Detection

### Purpose
Deliver the first headline capability: answer "what rights are available for Title X in territory Y on platform Z after date D?" and surface conflicts/holdbacks. After this phase rights managers and external systems can query availability in real time.

### Tasks

#### 4.1 — Avails query service + API

**What**: Multi-dimensional avails search.

**Design**: `services/avails.py`:
```python
@dataclass
class AvailsQuery:
    title_id: UUID | None
    territory: str | None        # single a2 or macro
    platform: str | None
    fmt: str | None
    language: str | None
    right_type: str | None
    as_of: date | None           # default today
@dataclass
class AvailsResult:
    grant_id: UUID; right_type: str; status: str
    territories: list[str]; platforms: list[str]; languages: list[str]
    window_start: date; window_end: date | None
    exclusivity: str | None; contract_ref: str; licensee: str
    availability: Literal["available","encumbered","expired"]
```
Implements the avails SQL from Data Model 1 (active grants, territory `ANY`, window overlap), extended with platform/format/language array filters and holdback computation (`as_of < window_start + holdback_days ⇒ encumbered`).
`GET /v1/avails?title_id=&territory=&platform=&right_type=&as_of=` → `list[AvailsResult]`. Every call writes an `avails_queried` audit entry (provenance idea from Data Model 3).

**Testing**:
- `Unit: availability classification — active+in-window→available; holdback active→encumbered; window passed→expired`
- `Integration: grant DE/streaming 2026-2028 → query DE/streaming/2027 returns available`
- `Integration: query FR (not granted) → empty`
- `Integration: macro-territory grant (EU) → query DE returns it`
- `Integration: avails query writes audit_log entry`

#### 4.2 — Conflict & holdback detection

**What**: Detect overlapping exclusive grants and holdback violations.

**Design**: `detect_conflicts(title_id) -> list[Conflict]` implements the Data Model 1 self-join (same title + right_type, territory `&&`, window overlap, at least one exclusive). Runs (a) on demand via `GET /v1/titles/{id}/conflicts` and (b) automatically after each grant create/update; on detection writes a `conflict_detected` audit entry and sets `rights_grants.status` is left unchanged but conflict is reported.
```python
@dataclass
class Conflict:
    grant_a: UUID; grant_b: UUID; right_type: str
    overlap_territories: list[str]; overlap_window: tuple[date, date | None]
    severity: Literal["exclusive_overlap","holdback_breach"]
```

**Testing**:
- `Unit: two exclusive DE/streaming grants overlapping in time → one conflict`
- `Unit: exclusive + non-exclusive overlap → conflict (exclusive breached)`
- `Unit: disjoint territories → no conflict`
- `Unit: adjacent non-overlapping windows → no conflict`
- `Integration: creating a conflicting grant triggers auto-detection + audit row`

---

## Phase 5: Sales Ingestion, FX & Anomaly Detection

### Purpose
Bring in the revenue data that royalties are computed from. Build pluggable importers for DSP/retail formats, historical FX conversion, title-matching, and (deferred AI) anomaly flagging. After this phase the platform holds matched sales data ready for the royalty engine.

### Tasks

#### 5.1 — Sales data model + importer framework

**What**: `sales_data` partitioned table and a pluggable importer interface for ≥3 formats (MVP).

**Design**: Implement `sales_data` per Data Model 1 (range-partitioned by `report_period_start`, all indexes). Importer contract:
```python
class SalesImporter(Protocol):
    channel: str
    def sniff(self, sample: bytes, filename: str) -> bool: ...
    def parse(self, fh: IO[bytes]) -> Iterator[SalesLine]: ...

@dataclass
class SalesLine:
    title_identifier: str | None      # ISRC/ISBN/UPC from the report
    title_text: str | None
    territory: str; currency_reported: str
    units_sold: int; streams: int
    gross_revenue_cents: int; net_revenue_cents: int
    report_period_start: date; report_period_end: date
    source_line_number: int
```
MVP importers: `spotify` (Spotify-for-Artists CSV), `apple` (Apple Music sales CSV), `ddex_dsr` (DDEX DSR flat file), plus `csv_generic` (column-mapping driven). `POST /v1/sales/import` (multipart) creates an `import_batch_id`, enqueues a Celery ingestion task, returns `202 {batch_id}`. Partition auto-creation mirrors audit-log.

**Testing**:
- `Unit: each importer.sniff correctly identifies its sample, rejects others`
- `Unit: spotify importer parses fixture → SalesLine list with cents (no float drift)`
- `Fixture: DDEX DSR sample parses to expected row count`
- `Integration: POST import → 202, batch created, task enqueued (eager mode), rows land in correct partition`

#### 5.2 — Title matching + FX conversion

**What**: Match sales lines to titles by identifier and convert reported currency to org default at historical rates.

**Design**: `fx_rates(date, base, quote, rate NUMERIC(18,8), source, PRIMARY KEY(date,base,quote))` (new table, additive). `FxService.convert(cents, frm, to, on)` looks up the rate for `report_period_end` (nearest prior day fallback), sets `exchange_rate` + `converted_cents`. Matching: exact identifier → title; on miss, mark `is_matched=false` and leave for review. A nightly Celery task pre-fetches ECB rates.

**Testing**:
- `Unit: convert with exact-day rate; nearest-prior fallback; missing rate → raises FxUnavailable`
- `Integration: import with known ISRCs → is_matched=true, converted_cents populated`
- `Integration: unknown identifier → is_matched=false, surfaced via GET /v1/sales?matched=false`

#### 5.3 — Anomaly detection (statistical + AI explanation) [v1.1]

**What**: Flag under-reporting, missed channels, and unexpected drops on incoming batches.

**Design**: `services/ingestion/anomaly.py` runs after a batch matches: per (title, channel) compute trailing-median revenue; flag lines below `median * 0.4` as `under_reporting`, and flag channels present historically but absent this period as `missed_channel`. Sets `is_anomalous=true`, `anomaly_reason`. An optional AI pass (`ai_tasks.explain_anomaly`) calls the LLM for a human-readable explanation stored in `ai_suggestions(suggestion_type='anomaly_alert')`.

**Testing**:
- `Unit: revenue 40% below trailing median → flagged under_reporting`
- `Unit: channel missing vs history → flagged missed_channel`
- `Integration: batch with one suppressed title → anomaly row + ai_suggestion created (LLM mocked)`

---

## Phase 6: Royalty Engine

### Purpose
The second headline capability: compute royalties from matched sales using the configurable rule set, including escalators, splits, advances, recoupment, and minimum guarantees. After this phase the platform produces auditable per-payee calculation rows.

### Tasks

#### 6.1 — Royalty rules model + CRUD

**What**: `royalty_rules` table and `/v1/contracts/{id}/rules`.

**Design**: Implement `royalty_rules` per Data Model 1 (rule_type CHECK, `rate_pct`, `flat_amount_cents`, `escalation_json`, `split_pct`, `applies_to`, `recoup_from_advance`, `priority_order`, `dsl_expression`). Endpoints CRUD rules scoped to a contract.

**Testing**:
- `Integration: create percentage rule + escalation tiers → persisted with JSONB intact`
- `Unit: rule with rule_type='escalating' but no escalation_json → 422`

#### 6.2 — Waterfall calculation engine

**What**: Deterministic, auditable calculation from sales → calculation rows.

**Design**: `services/royalty/engine.py`:
```python
@dataclass
class CalcRequest:
    contract_id: UUID; period_start: date; period_end: date
@dataclass
class CalcLine:
    rule_id: UUID; payee_party_id: UUID; title_id: UUID | None
    gross_cents: int; net_cents: int; units: int
    rate_applied_pct: Decimal; royalty_cents: int
    advance_offset_cents: int; net_payable_cents: int
```
Algorithm (priority_order ascending):
1. Aggregate matched `sales_data` for the contract's granted titles within the period → gross/net/units/streams per title.
2. For each active rule, compute base royalty by `applies_to` (gross/net/units/streams) and `rate_pct`/`flat_amount_cents`; apply `escalation_json` tiers against cumulative units.
3. Apply `co_author_split`/`sub_publisher_split` (`split_pct`) to the computed royalty.
4. Recoupment: if `recoup_from_advance`, offset against `contracts.advance_cents - advance_recouped_cents`; update running recouped balance.
5. Minimum guarantee: if total payable for period < pro-rated MG, add a top-up line.
All money math in integer cents with banker's rounding at the final cent. Each `royalty_calculations` row (Data Model 1) is written `status='draft'` with `calculation_notes` capturing the rule chain (Data Model 3 audit-trail intent). `POST /v1/contracts/{id}/calculate` body `CalcRequest` → `202` (Celery) or `200` with results in dev.

**Testing**:
- `Unit: flat % of net → royalty = net * rate, correct cents`
- `Unit: escalator crosses threshold mid-period → tier rate applied to units above threshold`
- `Unit: co-author 50/50 split → two payee lines summing to total`
- `Unit: recoupment — earnings < remaining advance → net_payable=0, recouped balance increases`
- `Unit: recoupment — earnings > remaining advance → partial offset, remainder payable`
- `Unit: MG shortfall → top-up line; MG met → no top-up`
- `Unit: rounding — 33.333% of 100 cents across 3 splits sums to original (no lost cent)`
- `Integration: end-to-end contract+grants+rules+sales → expected CalcLines, draft calculations + audit rows`

#### 6.3 — Royalty rule DSL [v1.1]

**What**: A safe expression language for `rule_type='custom_dsl'`.

**Design**: `services/royalty/dsl.py` — a small grammar (parsed with `lark`) over a sandboxed AST. Variables: `gross, net, units, streams, retail, wholesale, period_index, cumulative_units`. Operators: `+ - * /`, comparisons, `if/else`, `min/max`, tiered helper `tier(cumulative_units, [(threshold, rate), ...])`. No attribute access, no calls outside the allowlist (no `eval`). Evaluator returns cents.

**Testing**:
- `Unit: "net * 0.15" → correct`
- `Unit: "if units > 100000 then net*0.2 else net*0.15" → branch selection`
- `Unit: malicious "__import__('os')" → ParseError (not in grammar)`
- `Unit: division by zero → DslEvaluationError, not crash`

---

## Phase 7: Statements & Revenue/Audit Reporting

### Purpose
Turn calculation rows into payee-facing statements (PDF + CSV) with multi-currency, balances, and recoupment, and surface revenue/audit reporting. After this phase the full chain `sales → calculation → statement` is complete and distributable.

### Tasks

#### 7.1 — Royalty calculations API + approval

**What**: Expose and approve calculation rows before statementing.

**Design**: Implement `royalty_calculations` per Data Model 1 already created in 6.2. Endpoints:
- `GET /v1/calculations?contract_id=&payee_id=&period_start=&status=`
- `POST /v1/calculations/{id}:approve` (draft → approved; auditor cannot approve — RBAC)
- `POST /v1/calculations/{id}:void` with reason.

**Testing**:
- `Integration: approve draft → status approved, audit row`
- `Integration: auditor role attempts approve → 403`
- `Integration: void approved with reason → status void, reason in audit changes`

#### 7.2 — Statement generation (PDF + CSV) + balances

**What**: `royalty_statements` with line detail, running balances, and rendered documents.

**Design**: Implement `royalty_statements` per Data Model 1 (`lines_json`, balances, status state machine `draft→approved→sent→acknowledged→paid|disputed`, unique `(org_id, statement_ref)`). `StatementService.build(payee_id, period)` aggregates approved calculations into `lines_json`, computes `previous_balance`→`closing_balance` (carrying recoupment), renders:
- PDF via WeasyPrint from `statement.html.j2` (currency-formatted per ISO 4217, locale-aware).
- CSV via `csv` module (one row per line).
Both uploaded to S3; `document_url` set. `POST /v1/statements:generate` body `{payee_id, period_start, period_end}` → Celery task → statement row. `POST /v1/statements/{id}:send` transitions and (later) emails/portals.

**Testing**:
- `Unit: balance carry — previous 1000, royalties 500, paid 0 → closing 1500`
- `Unit: multi-currency lines grouped and totalled per currency`
- `Integration: generate from approved calcs → PDF+CSV in S3, lines_json matches calc rows`
- `Integration: CSV line count == calculation count; sum(royalty_cents) matches statement total`
- `E2E: contract→grants→rules→sales→calculate→approve→generate → downloadable PDF, exit ok`

#### 7.3 — Revenue tracking & audit reporting

**What**: Revenue-vs-guarantee tracking and discrepancy reporting.

**Design**: `GET /v1/reports/revenue?period=&group_by=title|channel|territory|payee` runs the Data Model 1 aggregation queries. `GET /v1/reports/discrepancies` compares reported sales totals vs received-payment totals per contract and flags shortfalls against MG. `GET /v1/audit?entity_type=&entity_id=&from=&to=` exposes the audit trail (auditor-readable).

**Testing**:
- `Integration: revenue report group_by=channel → totals match seeded sales`
- `Integration: contract under MG → discrepancy reported`
- `Integration: audit query returns chronological entries for an entity`

---

## Phase 8: Security, RBAC, Auth & API Hardening

### Purpose
Make the API safe for multi-tenant production: authentication, role-scoped authorisation, API keys, rate limiting, and OWASP API Top-10 hardening. After this phase every endpoint enforces tenant + role scoping and the OpenAPI contract is fuzz-tested.

### Tasks

#### 8.1 — Authentication (OIDC + JWT + API keys)

**What**: Login, token issuance/verification, OIDC SSO, and machine API keys.

**Design**: `security/auth.py` — password login (`argon2`) issuing JWT (RFC 7519: `sub`, `org_id`, `role`, `exp`); OIDC code flow via Authlib mapping `sub`→`users.external_subject`; `api_keys(id, org_id, hashed_key, name, scopes TEXT[], last_used_at, revoked)` table for machine clients (avails/rights integrations). `get_current_principal()` dependency resolves either a bearer JWT or `X-API-Key`.

**Testing**:
- `Unit: JWT encode/decode round-trip; expired token → 401`
- `Integration: login → token → authorised request succeeds`
- `Integration: tampered JWT signature → 401`
- `Integration: valid API key with avails scope hits /v1/avails; lacks scope on /v1/contracts → 403`

#### 8.2 — RBAC + tenant scoping

**What**: Role→permission matrix and enforced `org_id` scoping (data scoping per role, per README).

**Design**: `security/rbac.py` defines a matrix:
| role | titles | contracts | rights | royalties | statements | sales | audit |
|------|--------|-----------|--------|-----------|------------|-------|-------|
| admin | CRUD | CRUD | CRUD | CRUD | CRUD | CRUD | R |
| rights_manager | CRUD | CRUD | CRUD | R | R | R | R |
| royalty_accountant | R | R | R | CRUD | CRUD | CRUD | R |
| licensee | R(own) | R(own) | R(own) | – | – | C(own sales) | – |
| payee | – | – | – | R(own) | R(own) | – | – |
| auditor | R | R | R | R | R | R | R |
`require(perm)` dependency + row-level filters (`licensee`/`payee` see only rows where they are a party).

**Testing**:
- `Unit: matrix lookup for each (role, resource, action)`
- `Integration: payee GET /v1/statements → only own statements`
- `Integration: licensee cannot POST /v1/contracts → 403`
- `Integration: cross-tenant access to any entity → 404 (not 403, to avoid leak)`

#### 8.3 — Rate limiting & OWASP hardening

**What**: Per-key rate limits, payload caps, security headers, and contract fuzzing.

**Design**: Redis token-bucket middleware keyed by principal (defaults: 600/min authed, 60/min unauth). Request body size cap; strict CORS allowlist; security headers (HSTS, X-Content-Type-Options); mass-assignment prevented by explicit Pydantic models (OWASP API3); object-level auth enforced via 8.2 (OWASP API1). Schemathesis runs against `/openapi.json` in CI.

**Testing**:
- `Integration: exceed limit → 429 with Retry-After`
- `Integration: oversized body → 413`
- `Schemathesis: no 5xx across generated cases; auth required on protected paths`
- `bandit: no high-severity findings`

---

## Phase 9: Web Dashboard & Self-Service Portal

### Purpose
Provide the human UI: an internal dashboard for rights/royalty teams and a self-service portal for licensees/payees with e-signature. After this phase non-technical users can operate the platform end-to-end.

### Tasks

#### 9.1 — Internal dashboard

**What**: Next.js app for catalogue, avails, contracts, royalties, statements.

**Design**: Next.js 15 App Router + shadcn/ui + TanStack Query against the REST API. Pages: catalogue grid (server component, search/filter), title detail (contributors, grants, conflicts banner), avails explorer (territory/platform/date filters → results table), contract detail with status timeline, royalty run + calculation review, statement list/preview. Auth via OIDC; tokens in httpOnly cookies.

**Testing**:
- `E2E (Playwright): search catalogue → open title → see grants + conflict badge`
- `E2E: run avails query in UI → results match API`
- `Component: calculation review table renders cents as formatted currency`

#### 9.2 — Licensee/payee portal + e-signature [v1.1]

**What**: Scoped external portal with sales submission, statement download, and contract e-sign.

**Design**: Portal routes gated to `licensee`/`payee` roles. Licensee: submit sales (upload to `/v1/sales/import` scoped to own contracts), view own avails. Payee: download statements, view balances, manage tax/payment details. E-signature via `EsignAdapter` (DocuSign/Adobe Sign) — `POST /v1/contracts/{id}:send-for-signature` creates an envelope; webhook `POST /v1/webhooks/esign` updates `signature_json` and transitions contract to `executed` (eIDAS/ESIGN).

**Testing**:
- `Integration (mocked DocuSign): send-for-signature → envelope created, status pending`
- `Integration: signed webhook → signature_json set, contract executed, audit row`
- `E2E: payee logs in → downloads own statement PDF; cannot see other payees`

---

## Phase 10: Industry Interoperability (DDEX / ONIX / EIDR / CWR)

### Purpose
Deliver the stated technical advantage: first-class import/export of the open standards incumbents support only partially. After this phase catalogues and sales flow in/out via DDEX, ONIX, EIDR, and CWR.

### Tasks

#### 10.1 — DDEX ERN import & DSR ingestion

**What**: Import release metadata (ERN) and sales (DSR) as a native importer.

**Design**: `services/interop/ddex_ern.py` parses ERN XML → titles + identifiers + contributors. The `ddex_dsr` importer (started in 5.1) is completed to full DDEX DSR profile coverage. `POST /v1/interop/ddex/ern` (XML body) → created/updated titles.

**Testing**:
- `Fixture: ERN sample → expected titles with ISRC + contributors`
- `Fixture: DSR sample → SalesLines with correct territories/currencies`
- `Integration: re-import same ERN → idempotent upsert by identifier`

#### 10.2 — ONIX 3.0 + EIDR + CWR

**What**: ONIX book import/export, EIDR lookup, CWR export.

**Design**:
- `onix.py`: parse ONIX 3.0 product records → titles (ISBN, territory rights); export titles → ONIX. `POST /v1/interop/onix` and `GET /v1/interop/onix?title_id=`.
- `eidr.py`: client to EIDR REST API to resolve/enrich `eidr_id` for audiovisual titles.
- `cwr.py`: export controlled musical works as a CWR registration file (publisher/writer IPI, shares).

**Testing**:
- `Fixture: ONIX product → title with ISBN-13 + territory rights`
- `Unit: title → ONIX export validates against ONIX schema`
- `Integration (mocked EIDR): enrich title by ISAN → eidr_id populated`
- `Fixture: CWR export — writer/publisher shares sum to 100% per work`

---

## Phase 11: AI Features

### Purpose
Ship the AI-native differentiators that no surveyed incumbent offers: clause extraction, rights auto-classification, and natural-language avails. After this phase the platform turns unstructured contracts and questions into structured rights data.

### Tasks

#### 11.1 — LLM client abstraction + per-tenant isolation

**What**: A provider-agnostic LLM client enforcing tenant isolation and usage tracking.

**Design**: `services/ai/client.py` — `LLMClient.complete(system, user, *, schema=None, tenant_id) -> LLMResult`. Structured output via JSON-schema-constrained calls; records `tokens_used`/`llm_model` into `ai_suggestions`. Per IP notes: no cross-tenant prompts; no training on customer contracts; prompts tagged with `tenant_id` and logged.

**Testing**:
- `Unit (mocked): complete returns parsed structured object matching schema`
- `Unit: schema-violating model output → retried once then ValidationError`
- `Unit: usage recorded with correct token count`

#### 11.2 — Contract clause extraction [v1.1]

**What**: Extract royalty rates, territory definitions, exclusivity, and term dates from PDF contracts.

**Design**: `services/ai/clause_extraction.py` — pipeline: `pdfplumber` text extraction → chunk → LLM with extraction schema:
```json
{"royalty_terms":[{"payee":"","basis":"net_receipts","rate_pct":0,"escalators":[]}],
 "territories":[],"exclusivity":"","term":{"effective":"","expiry":""},"obligations":[]}
```
System prompt: "You are a rights-contract analyst. Extract only terms explicitly present. Return null for anything not stated. Never infer rates." Result stored in `contracts.clause_extraction_json` and surfaced as an `ai_suggestion(suggestion_type='clause_extraction')` for human verification before populating canonical fields. Triggered async on PDF upload (Celery `ai_tasks.extract_clauses`).

**Testing**:
- `Fixture (mocked LLM): sample contract text → expected royalty_terms, dates`
- `Integration: PDF upload → extract task → clause_extraction_json + ai_suggestion (pending)`
- `Unit: extractor never auto-applies to canonical fields (requires explicit accept)`

#### 11.3 — Rights auto-classification & NL avails [v1.1 / backlog]

**What**: Classify free-text grant descriptions into structured dimensions; answer NL avails questions.

**Design**:
- `classification.py`: text → `{right_type, territories[], platforms[], window}` suggestion for human confirmation.
- `nl_avails.py`: parse a question like "Can we stream Title X on TVOD in DACH after 2027-01-01?" into an `AvailsQuery`, run the Phase-4 service, and have the LLM phrase the answer; store as `ai_suggestion(suggestion_type='avails_answer')`. `POST /v1/avails/ask` body `{question}`.

**Testing**:
- `Unit (mocked): NL question → correct AvailsQuery params (territory=DACH→DE/AT/CH, right_type=tvod, as_of=2027-01-01)`
- `Integration: /v1/avails/ask → deterministic avails result + phrased answer`
- `Unit: classification suggestion is advisory only (never mutates grants)`

---

## Phase 12: Advanced / Backlog (Cryptographic Audit, MCP, Smart-Contract Disbursement)

### Purpose
Optional, opt-in differentiators for dispute-heavy and AI-agent use cases. Each is independent and additive.

### Tasks

#### 12.1 — Cryptographic audit trail (opt-in)

**What**: Tamper-evident hash chain over `audit_log`.

**Design**: Add `audit_log.prev_hash`/`entry_hash` (additive migration); each entry hashes `(prev_hash || canonical(entry))` (SHA-256). `GET /v1/audit/verify` recomputes the chain and reports the first divergence. Optional periodic anchoring of the head hash to an external notary/blockchain via an adapter (kept abstract — no chain dependency in core).

**Testing**:
- `Unit: chain over N entries verifies; mutate one entry → verify reports break at that index`
- `Integration: verify endpoint on seeded chain → ok=true`

#### 12.2 — MCP server

**What**: Expose `rights_avails`, `royalty_calc`, and `contracts` to LLM clients via MCP.

**Design**: `mcp/server.py` (Python MCP SDK) — tools: `rights_avails(title, territory, right_type, as_of)`, `royalty_calc(contract_id, period)`; resource: `contracts/{id}` returning structured terms + clause extraction. Auth via API key with MCP scope; all calls tenant-scoped and audit-logged.

**Testing**:
- `Integration (MCP test client): rights_avails tool → same result as /v1/avails`
- `Integration: unauthenticated MCP call → rejected`

#### 12.3 — AI-training-data licensing module [backlog]

**What**: Treat `ai_training` rights as a first-class, queryable, provenance-tracked right type.

**Design**: Leverage existing `right_type='ai_training'` grants + ODRL export; add a provenance report `GET /v1/titles/{id}/ai-training-provenance` returning ODRL policies + chain-of-title (contributors, grants) suitable for AI-training-data licence attestation. Monitor C2PA / IETF AI-Preferences for emerging formats.

**Testing**:
- `Integration: title with ai_training grant → provenance report includes ODRL policy + contributors`
- `Unit: titles without ai_training grant → report marks "no AI-training rights cleared"`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Tenancy            ─── required by everything
    │
Phase 2: Catalogue & Parties             ─── requires P1
    │
Phase 3: Contracts & Rights Grants       ─── requires P2
    │
    ├── Phase 4: Avails & Conflicts       ─── requires P3
    │
    └── Phase 5: Sales Ingestion/FX/Anom. ─── requires P2 (titles); P3 for matching to grants
            │
            └── Phase 6: Royalty Engine   ─── requires P3 + P5
                    │
                    └── Phase 7: Statements & Reporting ─── requires P6

Phase 8: Security/RBAC/Hardening         ─── requires P1; should land before any public deploy
                                              (apply scoping retroactively to P2–P7 endpoints)

Phase 9: Web Dashboard & Portal          ─── requires P4, P7, P8
Phase 10: Interoperability (DDEX/ONIX…)  ─── requires P2 (+ P5 for DSR); can parallel P6/P7
Phase 11: AI Features                     ─── requires P3 (clause→contracts), P4 (NL avails)
Phase 12: Advanced/Backlog                ─── P12.1 requires P1 audit; P12.2 requires P4/P6
```

**Parallelism opportunities**
- After Phase 3: **Phase 4** and **Phase 5** can be developed concurrently.
- **Phase 8** can be built alongside Phases 4–7 (a security-minded team applies RBAC as endpoints are added rather than retrofitting).
- After Phase 5: **Phase 10** (interop importers/exporters) can proceed in parallel with **Phase 6/7**.
- **Phase 11** AI features are independent of Phase 9/10 and can run in parallel once Phase 3/4 exist.

**Estimated scope: large** (12 phases, full-stack multi-tenant platform with AI and interoperability surfaces). A credible MVP is **Phases 1–8** (API-only, RBAC-secured: catalogue, contracts, rights, avails+conflicts, sales+FX, royalty engine, statements). v1.1 adds Phases 9–11.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests for the phase pass; coverage on new business logic ≥ 85% (royalty engine ≥ 95% — money math).
3. `ruff check` and `ruff format --check` pass.
4. `mypy --strict src` passes for new/changed modules.
5. `bandit -r src` reports no high-severity findings.
6. `docker compose build` succeeds and the stack boots (`/healthz` green).
7. The feature works end-to-end (demonstrated by at least one integration or E2E test exercising the real path).
8. New config options are documented in `.env.example` and the README.
9. New API endpoints appear in the auto-generated OpenAPI 3.1 spec and pass Schemathesis without 5xx.
10. Alembic migrations are created, are reversible, and `upgrade head` from empty produces a schema matching the ORM (no autogenerate diff).
11. Every state-mutating operation writes an `audit_log` entry within the same transaction.
12. All data-access paths are `org_id`-scoped (tenant isolation verified by a cross-tenant negative test).
```
