# Diversity, Equity & Inclusion Platform — Phased Development Plan

> Project: 416-diversity-equity-inclusion-platform · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The database design adopts **Data Model Suggestion 1** (normalised relational PostgreSQL with Row-Level Security) as its backbone, incorporating the **JSONB-for-jurisdictional-variability** idea from Suggestion 3 (demographic taxonomies, survey instruments, HRIS payloads, and report schemas stored as JSONB) and the **append-only audit log** principle from Suggestion 2 (every access to and mutation of special category data is recorded immutably).

The core value proposition: an AI-native, open-source, privacy-first platform unifying the full DEI lifecycle — workforce demographic analytics, regression-based pay equity, inclusion sensing, programme management, and CSRD/ESRS S1 + EEO-1 compliance reporting — built so that GDPR Article 9 handling, statistical rigour, and EU AI Act high-risk-system obligations are foundational constraints rather than configuration layers.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | Python 3.12 | The heart of the product is statistics (pay equity regression, significance testing) and AI (root-cause analysis, NLQ, sentiment). Python's `statsmodels`/`scikit-learn`/`scipy` ecosystem is unmatched for this and avoids a polyglot split between API and analytics. |
| API framework | FastAPI | Native OpenAPI 3.1 generation (a `standards.md` requirement), Pydantic v2 validation for JSON-Schema-described DEI payloads, async support for HRIS sync and LLM calls, first-class dependency injection for per-request RLS context. |
| Database | PostgreSQL 16 | Suggestion 1's choice. Row-Level Security enforces tenant + demographic isolation at the engine level (defence-in-depth for Article 9); JSONB handles jurisdictional taxonomy variability; generated columns, declarative partitioning, and `pgcrypto` are all used. |
| ORM / DB access | SQLAlchemy 2.0 (async) + Alembic | Explicit control over session-level `SET app.current_org_id` / `app.role` GUCs required for RLS. Alembic gives versioned, backward-compatible migrations. |
| Demographic encryption | Application-level AES-256-GCM via envelope encryption, key in HashiCorp Vault (or KMS) | Article 9 special category fields encrypted before storage; key separation means a DB dump alone cannot reveal demographics. |
| Task queue | Celery + Redis | Pay equity runs, survey theme extraction, HRIS syncs, and LLM analyses are long-running/async. Celery gives retries, scheduling (beat) for periodic syncs, and result tracking with run state machines. |
| LLM integration | Provider-abstracted client (OpenAI / Anthropic / self-hosted vLLM) behind an `LLMProvider` interface | Self-hosted deployments (data residency) need a local model option; the abstraction also isolates EU AI Act documentation/logging to one module. |
| HRIS integration | Merge unified HRIS API (primary) + native BambooHR/ADP adapters behind a common `HRISConnector` interface | `standards.md` explicitly recommends a unified API to avoid building 70+ connectors. Adapter pattern lets large customers use direct connectors. |
| Identity / auth | OAuth 2.0 + OIDC (Authlib), SAML 2.0 (python3-saml), SCIM 2.0 server | Enterprise SSO requires both OIDC and SAML; SCIM 2.0 ingests roster lifecycle events automatically. |
| Frontend | Next.js 15 (App Router, React, TypeScript) + shadcn/ui + Recharts | Dashboard-heavy product (representation, pay equity scorecards, heatmaps, manager views). Server Components for fast initial dashboard loads; Recharts for demographic visualisations. |
| Statistical engine | statsmodels (OLS pay-equity regression), scipy.stats (significance, k-anonymity checks) | Regression with explicit, inspectable coefficients is required for legal defensibility (EU Pay Transparency joint pay assessment). |
| Containerisation | Docker + docker-compose (dev), Helm chart (prod self-host) | Self-hosted/hybrid is a core deployment mode for data residency. |
| Testing | pytest (+ pytest-asyncio), Playwright (frontend e2e), Schemathesis (API contract from OpenAPI) | Schemathesis fuzzes the generated OAS 3.1 spec; Playwright covers dashboard flows. |
| Code quality | ruff (lint+format), mypy (strict), pre-commit | Strict typing matters in a compliance product where payload shapes are contractual. |
| Package manager | uv (Python), pnpm (frontend) | Fast, reproducible lockfiles. |
| Reporting / exports | openpyxl (EEO-1 XLSX/CSV), WeasyPrint (PDF), Jinja2 (CSRD/GRI narrative templates) | Regulators consume CSV/XLSX (EEO-1) and PDF (board/ESG); Jinja2 templates drive LLM-assisted narrative generation. |

### Project Structure

```
dei-platform/
├── pyproject.toml
├── uv.lock
├── Dockerfile
├── docker-compose.yml
├── helm/                              # production self-host chart
├── alembic.ini
├── migrations/                        # Alembic versioned migrations
├── openapi/                           # exported OAS 3.1 snapshot (CI-checked)
├── src/
│   └── dei/
│       ├── main.py                    # FastAPI app factory
│       ├── config.py                  # Pydantic Settings (env-driven)
│       ├── db/
│       │   ├── session.py             # async engine, RLS session context
│       │   ├── base.py                # declarative base, mixins (TenantMixin, TimestampMixin)
│       │   ├── rls.py                  # SET app.current_org_id / app.role helpers
│       │   └── models/                # SQLAlchemy models (organisation, employee, ...)
│       ├── crypto/
│       │   ├── envelope.py            # AES-256-GCM encrypt/decrypt
│       │   └── vault.py               # key provider abstraction
│       ├── auth/
│       │   ├── oidc.py, saml.py, scim.py
│       │   ├── rbac.py                # roles, permission checks
│       │   └── deps.py                # FastAPI dependencies (current_user, current_org, require_role)
│       ├── domain/
│       │   ├── demographics/          # taxonomy registry, consent, legal basis
│       │   ├── workforce/             # headcount/hiring/promotion/attrition analytics
│       │   ├── payequity/             # regression engine, runs, results
│       │   ├── surveys/               # instruments, anonymity, k-anonymity, cross-tabs
│       │   ├── programmes/            # mentorship matching, ERG
│       │   ├── goals/                 # OKR-style targets + progress
│       │   └── stats/                 # significance testing, k-anonymity primitives
│       ├── ai/
│       │   ├── provider.py            # LLMProvider interface + implementations
│       │   ├── rootcause.py           # automated root-cause analysis
│       │   ├── nlq.py                 # natural language → query plan
│       │   ├── coaching.py            # manager coaching prompts
│       │   ├── sentiment.py           # survey open-text themes
│       │   ├── narrative.py           # ESG/CSRD narrative generation
│       │   └── governance.py          # EU AI Act logging, model cards, human-oversight gates
│       ├── integrations/
│       │   ├── hris/                  # HRISConnector interface, Merge/BambooHR/ADP adapters
│       │   └── notify/                # Slack/Teams pay-decision prompts
│       ├── reporting/
│       │   ├── eeo1.py, csrd_esrs_s1.py, gri405.py, bloomberg_gei.py, pay_transparency.py
│       │   └── templates/             # Jinja2 narrative templates
│       ├── api/
│       │   └── v1/                    # routers per domain
│       ├── workers/
│       │   ├── celery_app.py
│       │   └── tasks/                 # payequity_run, hris_sync, survey_themes, reports
│       └── audit/
│           └── log.py                 # append-only audit writer
├── tests/
│   ├── unit/
│   ├── integration/                   # mocked HRIS/LLM; real Postgres via testcontainers
│   ├── e2e/
│   └── fixtures/                      # sample orgs, employees, comp, surveys (synthetic)
└── frontend/
    ├── package.json
    └── src/app/                       # Next.js App Router (dashboards, surveys, reports)
```

The structure is grouped by concern, not by phase. Every phase adds files within these directories without restructuring.

---

## Phase 1: Foundation, Tenancy & Privacy Core

### Purpose
Establish the project skeleton, the multi-tenant database with Row-Level Security, application-level encryption for special category data, and the append-only audit log. Nothing in this domain can be built safely without the privacy substrate first, because GDPR Article 9 handling is an architectural constraint (per `standards.md`). After this phase the system can store organisations and employees with tenant-isolated, encrypted, auditable demographic data.

### Tasks

#### 1.1 — Project scaffold, config, and DB session with RLS context

**What**: Stand up the FastAPI app, Pydantic settings, async SQLAlchemy engine, and a request-scoped session that sets PostgreSQL RLS GUCs.

**Design**:
```python
# config.py
class Settings(BaseSettings):
    database_url: str                       # postgresql+asyncpg://...
    redis_url: str = "redis://localhost:6379/0"
    vault_addr: str | None = None
    encryption_master_key: SecretStr        # dev fallback if Vault absent
    llm_provider: Literal["openai","anthropic","vllm","none"] = "none"
    environment: Literal["dev","staging","prod"] = "dev"
    model_config = SettingsConfigDict(env_prefix="DEI_", env_file=".env")

# db/rls.py
@asynccontextmanager
async def rls_session(session: AsyncSession, org_id: UUID, role: str):
    await session.execute(text("SET LOCAL app.current_org_id = :o"), {"o": str(org_id)})
    await session.execute(text("SET LOCAL app.role = :r"), {"r": role})
    yield session   # all queries within run under RLS for this org+role
```
`SET LOCAL` ties the GUC to the transaction so it cannot leak across pooled connections. The FastAPI dependency `current_org`/`current_role` wraps every request in `rls_session`.

**Testing**:
- `Unit: Settings loads from env with DEI_ prefix → correct typed fields, secrets not logged`
- `Integration (real Postgres): two orgs inserted; query under org A's RLS context → only org A rows returned`
- `Integration: SET LOCAL scoped to transaction → after commit, new transaction without GUC sees zero rows (deny-by-default)`

#### 1.2 — Core tenancy schema + RLS policies (Alembic migration 0001)

**What**: Create `organisation`, `department`, `team`, `employee`, `employment_history` tables with `ENABLE ROW LEVEL SECURITY` and `tenant_isolation` policies.

**Design**: Use the DDL from `data-model-suggestion-1.md` for `organisation`, `department`, `employee`. Add `team` (FK to department) and a temporal `employment_history`:
```sql
CREATE TABLE employment_history (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    employee_id UUID NOT NULL REFERENCES employee(id),
    event_type TEXT NOT NULL CHECK (event_type IN ('hire','promotion','transfer','termination','leave_start','leave_end')),
    event_date DATE NOT NULL,
    from_department_id UUID, to_department_id UUID,
    from_level INT, to_level INT
);
```
Every tenant-scoped table gets:
```sql
ALTER TABLE <t> ENABLE ROW LEVEL SECURITY;
ALTER TABLE <t> FORCE ROW LEVEL SECURITY;   -- applies even to table owner
CREATE POLICY tenant_isolation ON <t>
  USING (organisation_id = current_setting('app.current_org_id')::UUID);
```
`organisation.data_residency` (`'eu'|'us'|'uk'|'apac'`) and `gdpr_legal_basis` drive later routing/compliance logic.

**Testing**:
- `Integration: cross-tenant SELECT/UPDATE/DELETE blocked by RLS (0 rows affected)`
- `Integration: FORCE RLS prevents superuser-owner bypass`
- `Unit: employment_history CHECK rejects unknown event_type`

#### 1.3 — Encrypted demographics + consent/legal-basis tracking

**What**: Implement `employee_demographic` with application-level field encryption, separate RLS + role gate, and consent metadata.

**Design**: `employee_demographic` per Suggestion 1, but raw demographic columns stored as `BYTEA` ciphertext; plaintext only via the `crypto.envelope` module.
```python
# crypto/envelope.py
def encrypt_field(plaintext: str, dek: bytes) -> bytes  # AES-256-GCM, nonce prepended
def decrypt_field(ciphertext: bytes, dek: bytes) -> str
# DEK fetched per-org from Vault; master key wraps DEKs (envelope encryption)
```
The demographic RLS policy additionally requires `current_setting('app.role') IN ('dei_analyst','hr_admin','system')`. `legal_basis` is `NOT NULL` (`'explicit_consent'|'employment_law'|'public_interest'`); `data_source` constrained to `('self_reported','hris_import','inferred')`. Demographic taxonomy values are validated against a per-jurisdiction registry (Task 1.5) rather than hardcoded enums.

**Testing**:
- `Unit: encrypt→decrypt round-trips; tampered ciphertext (flipped byte) → GCM auth failure`
- `Integration: query under role 'manager' → demographic rows hidden by RLS; under 'dei_analyst' → visible`
- `Unit: missing legal_basis → IntegrityError`
- `Unit: DB dump (raw BYTEA) is not human-readable without DEK`

#### 1.4 — Append-only audit log

**What**: Record every read and write of demographic/compensation data immutably.

**Design**:
```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    organisation_id UUID NOT NULL,
    actor_id UUID, actor_role TEXT NOT NULL,
    action TEXT NOT NULL CHECK (action IN ('read','create','update','delete','export')),
    entity TEXT NOT NULL, entity_id UUID,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    request_id UUID, detail JSONB
);
REVOKE UPDATE, DELETE ON audit_log FROM PUBLIC;   -- append-only
```
A FastAPI middleware + an `audit.log.record()` helper write entries; demographic-table reads emit `action='read'`. Optional hash-chain (`prev_hash`) for tamper evidence.

**Testing**:
- `Integration: reading employee_demographic emits audit_log row with action='read', correct entity_id`
- `Integration: UPDATE/DELETE on audit_log raises insufficient privilege`
- `Unit: hash-chain link verifies; altering a past row breaks the chain`

#### 1.5 — Jurisdictional demographic taxonomy registry (JSONB)

**What**: A configurable registry of valid demographic categories per jurisdiction, so EU/US/UK ethnicity and gender taxonomies differ without schema migrations (Suggestion 3 idea).

**Design**:
```sql
CREATE TABLE demographic_taxonomy (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    jurisdiction CHAR(2) NOT NULL,           -- 'US','GB','DE', or 'XX' default
    dimension TEXT NOT NULL,                 -- 'gender','ethnicity','disability',...
    version TEXT NOT NULL,
    categories JSONB NOT NULL,               -- [{code, label, eeo1_mapping?, ...}]
    active BOOLEAN NOT NULL DEFAULT true
);
```
Seed US EEO-1 7-category race/ethnicity (2024 expansion per `standards.md`), EU/UK common categories. A `TaxonomyValidator.validate(jurisdiction, dimension, code)` is used by demographic ingestion.

**Testing**:
- `Unit: validate('US','ethnicity','two_or_more') → ok; unknown code → error listing valid codes`
- `Unit: jurisdiction fallback to 'XX' default when specific jurisdiction missing`

---

## Phase 2: Authentication, RBAC, SSO & SCIM

### Purpose
Enterprise buyers require OIDC/SAML SSO and SCIM provisioning (`standards.md`). RBAC gates access to special category data and is the application-layer half of the Article 9 defence (RLS is the DB half). After this phase, users authenticate via enterprise IdPs, roster lifecycle events flow in via SCIM, and every endpoint enforces role-based permissions.

### Tasks

#### 2.1 — RBAC model and permission enforcement

**What**: Define roles, map them to permissions, and provide a `require_role`/`require_permission` FastAPI dependency.

**Design**:
```python
class Role(StrEnum):
    SYSTEM="system"; HR_ADMIN="hr_admin"; DEI_ANALYST="dei_analyst"
    MANAGER="manager"; EXECUTIVE="executive"; EMPLOYEE="employee"
PERMISSIONS = {
    Role.DEI_ANALYST: {"demographics:read","payequity:run","survey:manage","report:generate"},
    Role.MANAGER: {"team_analytics:read","coaching:read"},  # aggregated, k-anon enforced
    ...
}
def require_permission(perm: str) -> Callable: ...   # dependency raising 403
```
The user's role is also injected into the RLS GUC (`app.role`) so DB and app layers agree.

**Testing**:
- `Unit: manager lacks demographics:read → 403`
- `Integration: dei_analyst calls /payequity/run → 200; employee → 403`
- `Integration: role in JWT must match role set on RLS session (no privilege mismatch)`

#### 2.2 — OIDC + SAML SSO

**What**: Implement OIDC (Authlib) and SAML 2.0 (python3-saml) login flows producing platform session JWTs.

**Design**: `/auth/oidc/login`, `/auth/oidc/callback`, `/auth/saml/acs`, `/auth/saml/metadata`. Per-org IdP config in `organisation_idp` (JSONB for entity_id, ACS URL, certs). Session JWT carries `sub`, `org_id`, `role`, `exp`.

**Testing**:
- `Integration (mocked IdP): valid OIDC code → JWT with correct org_id/role`
- `Integration: SAML assertion with invalid signature → 401, no session`
- `Unit: expired JWT → 401`

#### 2.3 — SCIM 2.0 server

**What**: Implement `/scim/v2/Users` and `/scim/v2/Groups` for automated provisioning.

**Design**: Standard SCIM 2.0 (RFC 7644) CRUD. `POST /Users` creates an `employee` + user; `PATCH` (active=false) marks termination and writes an `employment_history` `termination` event. Secured with OAuth 2.0 bearer tokens.

**Testing**:
- `Integration: POST /scim/v2/Users → employee + user created, audit logged`
- `Integration: PATCH active=false → employee.status='terminated', employment_history event written`
- `Unit: SCIM error responses conform to RFC 7644 schema`

---

## Phase 3: HRIS Integration & Compensation Ingestion

### Purpose
The platform is only useful with real workforce data. This phase builds the `HRISConnector` abstraction, the Merge adapter (primary), direct BambooHR/ADP adapters, and the ETL that lands data in staging then normalises it into core tables including temporal `compensation` (required for pay equity). After this phase, an org can connect its HRIS and have employees, org structure, demographics, and compensation synced.

### Tasks

#### 3.1 — HRISConnector interface + staging tables

**What**: Define a connector contract and raw staging tables (JSONB payloads, Suggestion 3) decoupled from normalisation.

**Design**:
```python
class HRISConnector(Protocol):
    async def list_employees(self, since: datetime|None) -> AsyncIterator[RawEmployee]: ...
    async def list_compensation(self, since: datetime|None) -> AsyncIterator[RawComp]: ...
    async def list_org_units(self) -> AsyncIterator[RawOrgUnit]: ...
```
```sql
CREATE TABLE hris_staging (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    source TEXT NOT NULL,          -- 'merge','bamboohr','adp'
    record_type TEXT NOT NULL,     -- 'employee','compensation','org_unit'
    external_id TEXT NOT NULL,
    payload JSONB NOT NULL,
    ingested_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed BOOLEAN NOT NULL DEFAULT false
);
```

**Testing**:
- `Unit: connector Protocol conformance check for each adapter`
- `Integration: staging insert idempotent on (org, source, external_id, record_type)`

#### 3.2 — Merge adapter + BambooHR/ADP adapters

**What**: Implement adapters mapping each source's schema to `RawEmployee`/`RawComp`/`RawOrgUnit`.

**Design**: Merge adapter uses Merge's normalised HRIS model (one mapping). BambooHR adapter (API-key Basic Auth) reads custom fields for demographics. ADP adapter (OAuth 2.0, partner cert) reads compensation. Each adapter handles its auth and rate limits (Workday-via-Merge respects 10 req/s).

**Testing**:
- `Integration (mocked Merge API): paginated employee fetch → RawEmployee list with normalised fields`
- `Integration (mocked BambooHR): custom field 'ethnicity' mapped into demographic payload`
- `Unit: rate-limit backoff on 429`

#### 3.3 — Normalisation ETL + compensation temporal model

**What**: A Celery task transforms staging rows into `employee`, `employee_demographic` (encrypted, taxonomy-validated), `compensation`, and `employment_history`.

**Design**: Use Suggestion 1's `compensation` DDL (generated `total_comp`, `pay_decision_type`). Inferred demographics from HRIS get `data_source='inferred'` and are flagged for EU exclusion (GDPR restricts inferred ethnicity — `standards.md`). The ETL diffs against current state to emit `employment_history` events (promotion = level change, transfer = dept change).

**Testing**:
- `Integration: staging → normalised; level increase produces 'promotion' history event`
- `Unit: inferred ethnicity in EU-residency org → rejected/quarantined`
- `Integration: re-running ETL on unchanged data is a no-op (no duplicate history)`

#### 3.4 — Sync scheduling & status

**What**: Celery-beat periodic incremental syncs with a run state machine and surfaced status.

**Design**: `hris_sync_run(status: 'pending'|'running'|'completed'|'failed', records_processed, error)`. `GET /api/v1/integrations/hris/status` returns last run per source.

**Testing**:
- `Integration: failed adapter call → run.status='failed', error captured, retried per Celery policy`
- `Integration: incremental sync passes 'since' = last successful run timestamp`

---

## Phase 4: Statistical Core — Workforce Analytics & Significance Testing

### Purpose
This is the analytical heart. Build the demographic analytics engine (headcount, hiring, promotion, attrition by dimension) with statistical significance testing and intersectional cross-tabulation, all enforcing k-anonymity confidentiality thresholds. After this phase the representation dashboard backend works against real data with statistically sound outputs — the MVP table-stakes capability.

### Tasks

#### 4.1 — k-anonymity and significance primitives

**What**: Shared stats utilities used by every analytics/survey query.

**Design**:
```python
def enforce_k_anonymity(groups: dict[str,int], k: int = 5) -> dict[str,int|Literal["suppressed"]]:
    # any group with count < k returned as "suppressed"
def two_proportion_z_test(success_a,n_a,success_b,n_b) -> SignificanceResult  # p, z, ci
def chi_square_independence(contingency: list[list[int]]) -> SignificanceResult
@dataclass
class SignificanceResult: statistic: float; p_value: float; significant: bool; ci_low: float; ci_high: float
```
`k` default 5, configurable per survey/org (`anonymity_threshold`). Suppressed cells are never returned with raw counts.

**Testing**:
- `Unit: group of 3 with k=5 → "suppressed"`
- `Unit: known 2-prop input → matches scipy reference p-value within 1e-6`
- `Unit: chi-square on balanced table → not significant; on skewed table → significant`

#### 4.2 — Representation metrics engine

**What**: Compute headcount/hiring/promotion/attrition rates segmented by demographic dimension with significance vs. a baseline.

**Design**:
```python
def representation(org_id, dimension, filters, period) -> RepresentationReport
@dataclass
class DimensionBreakdown: category: str; count: int|Literal["suppressed"]; pct: float|None; vs_baseline: SignificanceResult|None
```
Attrition rate = terminations / avg headcount over period (from `employment_history`). Promotion rate = promotions / eligible. Every breakdown passes through `enforce_k_anonymity`.

**Testing**:
- `Unit (fixture org): attrition by gender matches hand-computed values`
- `Integration: small subgroup suppressed in output`
- `Unit: promotion-rate gap flagged significant when p<0.05`

#### 4.3 — Intersectional cross-tabulation

**What**: Multi-dimension breakdowns (e.g., ethnicity × gender × level), a `features.md` differentiator.

**Design**: `intersectional(org_id, dimensions: list[str], metric) -> NestedBreakdown`. Combinatorial explosion bounded by k-anonymity suppression and a max-dimensions guard (default 3).

**Testing**:
- `Unit: ethnicity×gender produces correct cells; cells <k suppressed`
- `Unit: >3 dimensions → ValueError`

#### 4.4 — Analytics API endpoints

**What**: REST endpoints feeding dashboards.

**Design**: `GET /api/v1/analytics/representation`, `/attrition`, `/promotion`, `POST /analytics/intersectional`. Responses are Pydantic models → OAS 3.1. Demographic detail requires `demographics:read`; managers get only their team aggregates (k-anon enforced).

**Testing**:
- `Integration: dei_analyst gets full breakdown; manager gets team-scoped, suppressed small cells`
- `Schemathesis: endpoints conform to generated OAS 3.1 spec`

---

## Phase 5: Pay Equity Engine

### Purpose
Regression-based pay equity is a top MVP feature and the basis for EU Pay Transparency Directive (deadline 7 June 2026) compliance. This phase builds the regression model, run lifecycle, per-employee residuals, and gap detection, producing legally defensible, inspectable results. After this phase an org can run a pay equity analysis and see explained vs. unexplained gaps.

### Tasks

#### 5.1 — Regression model

**What**: OLS model of `log(total_comp)` on controls + protected characteristic.

**Design**: `statsmodels.OLS`. Controls (configurable): `job_level`, `tenure_years`, `geography`, `department`. Protected characteristic (e.g., gender) added as the variable of interest; its coefficient is the adjusted gap. Returns coefficients, p-values, R², and per-employee predicted vs. actual.
```python
@dataclass
class PayEquityModel:
    adjusted_gap_pct: float; p_value: float; r_squared: float
    controls: list[str]; protected: str
    coefficients: dict[str, float]
```
Log-transform makes the protected coefficient interpretable as an approximate % gap (EU joint-pay-assessment threshold = 5%).

**Testing**:
- `Unit (synthetic data with injected 8% gap): recovered adjusted_gap_pct ≈ 8% ± tolerance`
- `Unit: no real gap → adjusted gap not significant`
- `Unit: perfectly collinear controls → handled (drop + warn), no crash`

#### 5.2 — Run lifecycle + results persistence

**What**: Async pay-equity runs with the Suggestion 1 `pay_equity_run`/`pay_equity_result` schema.

**Design**: Celery task; `pay_equity_run.status` state machine `pending→running→completed|failed`. `summary_json` holds aggregate (no PII); `pay_equity_result` holds per-employee residual + `flag` (`within_range|underpaid|overpaid|review`). Underpaid flag when actual < predicted − threshold.

**Testing**:
- `Integration: POST /payequity/run enqueues; status transitions observed`
- `Integration: results flag correct employees as underpaid given known fixture`
- `Unit: summary_json contains no individual identifiers`

#### 5.3 — Real-time pay-decision monitoring + Slack/Teams alerts

**What**: On new compensation records, check whether the decision worsens a group gap and notify (Syndio-style in-workflow guidance).

**Design**: Trigger on `compensation` insert → lightweight incremental gap check → if a protected-group gap crosses threshold, post to Slack/Teams via `integrations/notify`. `GET /api/v1/payequity/alerts`.

**Testing**:
- `Integration (mocked Slack): new low offer widening gap → alert posted`
- `Integration: within-range decision → no alert`

---

## Phase 6: Inclusion Surveys & Sensing

### Purpose
Inclusion sensing (belonging, psychological safety, fair treatment, voice) complements demographic representation with lived experience — an MVP feature. The privacy challenge is keeping responses unlinkable to employees while still cross-tabulating by demographic. After this phase orgs can run validated pulse surveys with anonymity guarantees and demographic cross-tabs.

### Tasks

#### 6.1 — Survey instrument model + validated template

**What**: `survey`, `survey_question`, and a seeded scientifically-validated instrument across the four dimensions.

**Design**: Suggestion 1 `survey` DDL with `instrument_version` and `anonymity_threshold`. Seed a Likert (1–5) instrument; each question tagged `dimension`. Instrument definitions stored as JSONB so they evolve without migration.

**Testing**:
- `Unit: seeded instrument has questions covering all four dimensions`
- `Unit: anonymity_threshold default 5; configurable per survey`

#### 6.2 — Pseudonymous response collection (anonymity firewall)

**What**: Collect responses against a `respondent_pseudo_id` that is never linkable back to `employee`.

**Design**: Suggestion 1 `survey_response`/`survey_answer`. Demographics for cross-tab stored detached in `survey_response_demographic` with no FK to the response's identity beyond what k-anonymity allows; the employee→pseudo mapping is generated per-survey, one-way (HMAC with a per-survey key that is destroyed after distribution), so even an admin cannot re-link.

**Testing**:
- `Integration: submitted response has no recoverable employee link`
- `Unit: pseudo-id generation one-way (cannot invert)`
- `Security test: attempt to join survey_response to employee returns nothing`

#### 6.3 — Scoring, demographic cross-tabs, k-anonymity

**What**: Dimension scores overall and cross-tabbed by demographic, suppressing small cells.

**Design**: `survey_scores(survey_id, by: list[str]|None) -> SurveyScoreReport`. Cross-tabs reuse Phase-4 k-anonymity; any demographic cell below threshold is suppressed before results leave the DB layer.

**Testing**:
- `Unit (fixture): dimension means match hand calc`
- `Integration: demographic cell of 3 respondents suppressed`

#### 6.4 — Open-text theme extraction (AI)

**What**: LLM-driven theme/sentiment extraction from open-ended answers.

**Design**: `ai/sentiment.py` batches open-text answers (stripped of identifiers) to the `LLMProvider`, returns themes with counts + sentiment. Runs as a Celery task. Per EU AI Act, logged via `ai/governance.py`; output is decision-support only (no automated action on individuals).

**Testing**:
- `Integration (mocked LLM): themed output parsed into structured themes`
- `Unit: text passed to LLM contains no employee identifiers`

---

## Phase 7: Goals, Programme Management & Mentorship Matching

### Purpose
Goal tracking (MVP) and programme management (v1.1) round out the lifecycle that competitors split across separate tools. The DEI-aware mentorship matching engine is a Qooper-style differentiator. After this phase orgs set representation/pay/inclusion targets, track progress, and run mentorship + ERG programmes with outcome linkage.

### Tasks

#### 7.1 — Goals & progress tracking

**What**: OKR-style DEI goals with milestones and time-series progress tied to live metrics.

**Design**: Suggestion 1 `goal`/`goal_milestone`/`goal_progress`. A goal references a metric source (e.g., `representation:gender:leadership`); progress is computed from the Phase-4 engine on a schedule.

**Testing**:
- `Unit: goal target 40% leadership women, current 32% → progress 80% of way (relative) computed correctly`
- `Integration: nightly job appends goal_progress row from live metric`

#### 7.2 — Mentorship matching engine

**What**: DEI-aware matching on skills, career goals, and diversity dimensions.

**Design**: `match(participants) -> list[MentorshipPair]` scoring on skill-gap complementarity, goal alignment, and optional cross-demographic/cross-department pairing weights. Greedy + stable-matching fallback. Uses only consented demographic data.

**Testing**:
- `Unit: mentor with matching expertise to mentee gap scores higher`
- `Unit: cross-department weighting produces cross-dept pairs when enabled`
- `Unit: participant who did not consent to demographic use → matched without demographic signal`

#### 7.3 — ERG management & participation-to-outcome linkage

**What**: ERG groups, membership, event tracking, and causal-ish linkage of participation to promotion/retention.

**Design**: `erg`/`erg_membership`/`programme_participation` (Suggestion 1). A comparison report contrasts promotion/retention rates of participants vs. matched non-participants (propensity-style matching on level/tenure/dept), with k-anonymity.

**Testing**:
- `Integration: participants vs non-participants promotion-rate comparison computed with significance`
- `Integration: small ERG suppressed in reporting`

---

## Phase 8: Compliance Reporting & Exports

### Purpose
ESG/regulatory reporting is a near-term enterprise requirement and a key differentiator (`research.md`). This phase implements the report framework engine and the specific exports: EEO-1, CSRD/ESRS S1, GRI 405, Bloomberg GEI, and EU Pay Transparency. After this phase orgs generate audit-ready, stakeholder-ready compliance reports from underlying DEI data.

### Tasks

#### 8.1 — Report template engine + run lifecycle

**What**: Suggestion 1 `report_template`/`report_run` with framework-specific JSONB schemas and a draft→review→approved→submitted lifecycle.

**Design**: `ReportGenerator(framework).generate(org_id, period) -> ReportRun`. `data_snapshot` is aggregated + anonymised only. Framework registry maps `framework` → generator + export format.

**Testing**:
- `Unit: report_run lifecycle transitions valid; illegal transition rejected`
- `Integration: data_snapshot contains no individual PII`

#### 8.2 — EEO-1 Component 1 export

**What**: US EEO-1 CSV/XLSX with 10 job categories × 7 race/ethnicity (2024 expansion) × sex.

**Design**: `reporting/eeo1.py` maps employees to EEO-1 job categories and the taxonomy registry's `eeo1_mapping`; outputs the official grid via openpyxl.

**Testing**:
- `Integration (fixture org): EEO-1 grid totals reconcile to headcount`
- `Unit: 2024 expanded race/ethnicity categories present`

#### 8.3 — CSRD/ESRS S1 + GRI 405 + Bloomberg GEI + Pay Transparency

**What**: EU/ESG report generators aligned to the named standards.

**Design**: ESRS S1 covers S1-9 (gender distribution at top management, age distribution) and S1-12 (% employees with disabilities). GRI 405 gender breakdown by governance body/category + gender pay ratio. Bloomberg GEI five-pillar scoring. Pay Transparency: gender pay gap report; if gap >5% and unexplained, flag joint-pay-assessment workflow. PDF via WeasyPrint; narrative via Jinja2 + optional LLM (Phase 9).

**Testing**:
- `Unit: ESRS S1-9 datapoints populated; S1-12 disability % computed`
- `Unit: Pay Transparency gap >5% unexplained → joint_pay_assessment flag set`
- `Integration: GRI 405 gender pay ratio matches Phase-5 output`

---

## Phase 9: AI-Native Layer & Governance

### Purpose
The AI-native advantages — root-cause analysis, NLQ, manager coaching, narrative generation — are the core differentiator (`README.md`). Because these are EU AI Act high-risk employment systems, this phase ships them together with the governance, logging, and human-oversight controls (`standards.md`). After this phase non-technical users query in natural language, gaps are auto-explained, managers get coaching, and ESG narratives are drafted — all auditable.

### Tasks

#### 9.1 — AI governance substrate (EU AI Act / ISO 42001)

**What**: Logging, model cards, human-oversight gates, and bias checks wrapping every AI feature.

**Design**: `ai/governance.py` records each inference (feature, model, input hash, output, timestamp, org) to an immutable store; `requires_human_review` flag on any output influencing an individual decision; model cards documenting purpose/limitations. NIST AI RMF trustworthiness characteristics tracked.

**Testing**:
- `Integration: every AI call produces a governance log entry`
- `Unit: outputs flagged decision-influencing carry requires_human_review=True`

#### 9.2 — Automated root-cause analysis

**What**: AI identifies which department/manager/process drives an observed gap.

**Design**: `ai/rootcause.py` runs structured drill-downs (Phase-4 engine) across org units, ranks contributors by effect size + significance, and uses the LLM only to *narrate* the statistically-derived findings (numbers come from stats, not the LLM — avoids hallucinated metrics).

**Testing**:
- `Integration (fixture with planted gap in one dept): that dept ranked top contributor`
- `Unit: all numeric claims in narrative trace to a stats result (no fabricated figures)`

#### 9.3 — Natural language querying

**What**: Translate HR questions into safe, parameterised analytics queries.

**Design**: `ai/nlq.py` maps NL → a constrained query plan (allowlisted metrics/dimensions/filters), executed by the Phase-4 engine under the caller's RLS+RBAC. The LLM never emits raw SQL; only a validated plan object. k-anonymity still applies.

**Testing**:
- `Unit: "attrition for women in engineering last year" → correct query-plan object`
- `Security: prompt-injection attempt cannot escalate beyond caller's role/org or bypass suppression`

#### 9.4 — Manager coaching prompts

**What**: Contextualised inclusion guidance per manager based on team composition + sentiment, addressing the `features.md` underserved area.

**Design**: `ai/coaching.py` consumes the manager's k-anonymised team metrics + survey signals and produces guidance. Suppressed where team too small to protect anonymity. Advisory only; governance-logged.

**Testing**:
- `Integration: manager with small team → coaching suppressed (anonymity)`
- `Unit: coaching input is team-aggregated, never individual demographics`

#### 9.5 — ESG narrative generation

**What**: Draft CSRD/GRI narrative prose from the Phase-8 data snapshot.

**Design**: `ai/narrative.py` fills Jinja2 framework templates with snapshot figures and uses the LLM for prose; all figures injected from the snapshot, marked draft, human-review required before `submitted`.

**Testing**:
- `Integration: generated narrative figures match data_snapshot`
- `Unit: narrative report_run cannot reach 'submitted' without human approval`

---

## Phase 10: Frontend, Manager Views & Hardening

### Purpose
Deliver the dashboards and manager experience that make the platform usable by non-technical HR users and frontline managers, then harden for production (rate limiting, OAS publication, security review, deployment). After this phase the product is shippable end-to-end.

### Tasks

#### 10.1 — Dashboards (representation, pay equity, surveys, goals)

**What**: Next.js dashboards consuming the v1 API.

**Design**: Representation dashboard with drill-down (dept/manager/level/geography) and demographic heatmaps (Recharts); pay equity scorecard; survey dimension heatmaps; goal progress. Suppressed cells render as "insufficient data" (never raw counts). NLQ search bar wired to Phase-9 endpoint.

**Testing**:
- `Playwright e2e: analyst views representation, drills into a dept, sees significance markers`
- `Playwright: small subgroup shows "insufficient data", not a count`

#### 10.2 — Manager experience

**What**: Manager-scoped view with coaching prompts.

**Design**: Manager sees only their team's k-anonymised aggregates + coaching; no individual demographics. Enforced by RBAC (Phase 2) + RLS scoping.

**Testing**:
- `Playwright: manager cannot access org-wide demographic detail (UI + API 403)`

#### 10.3 — Production hardening

**What**: Rate limiting, OAS 3.1 publication, Helm chart, security/DPIA artefacts, data residency routing.

**Design**: Per-org rate limits; export `openapi/openapi.json` in CI and diff-check; Helm chart for self-host; `organisation.data_residency` routes to the correct regional DB instance; ship DPIA template + Article 9 legal-basis documentation; run dependency + secret scans.

**Testing**:
- `Integration: rate limit returns 429 past threshold`
- `CI: generated OAS matches committed snapshot (drift fails build)`
- `Integration: EU-residency org's data never written to non-EU instance`
- `Schemathesis: full API conforms to OAS 3.1`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation, Tenancy & Privacy Core   ─── required by everything
    │
Phase 2: Auth, RBAC, SSO & SCIM               ─── requires Phase 1
    │
Phase 3: HRIS Integration & Compensation      ─── requires Phase 1, 2
    │
Phase 4: Statistical Core (Analytics)         ─── requires Phase 3
    │
    ├── Phase 5: Pay Equity Engine             ─── requires Phase 4 (uses comp + stats)
    ├── Phase 6: Inclusion Surveys & Sensing   ─── requires Phase 4 (k-anon) ── can parallel Phase 5, 7
    └── Phase 7: Goals, Programmes, Mentorship ─── requires Phase 4 ── can parallel Phase 5, 6
            │
Phase 8: Compliance Reporting & Exports        ─── requires Phase 4, 5, 6 (pulls all metrics)
    │
Phase 9: AI-Native Layer & Governance          ─── requires Phase 4, 5, 6, 8 (narrates their outputs)
    │
Phase 10: Frontend, Manager Views & Hardening  ─── requires all (surfaces everything)
```

**Parallelism opportunities:**
- After Phase 4, **Phases 5, 6, and 7** can be developed concurrently — they depend on the shared statistical core but not on each other.
- Within Phase 9, root-cause (9.2), NLQ (9.3), coaching (9.4), and narrative (9.5) can be built in parallel once the governance substrate (9.1) exists.
- Frontend (Phase 10) work for a given domain can begin as soon as that domain's API (Phases 4–8) is stable.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit and integration tests pass (`pytest`), including mocked-dependency integration tests.
3. Real-dependency integration tests (Postgres via testcontainers) pass; RLS isolation tests pass.
4. `ruff` lint + format clean; `mypy --strict` clean.
5. Alembic migration(s) created, apply cleanly forward, and (where feasible) reverse.
6. The phase's feature works end-to-end against fixture data.
7. New config options documented in `config.py` and `.env.example`.
8. New API endpoints appear in the generated OAS 3.1 spec and pass Schemathesis contract tests.
9. Every read/write/export of special category (demographic) or compensation data emits an `audit_log` entry.
10. k-anonymity / confidentiality thresholds are enforced and tested for any new analytic output.
11. AI features (Phase 9) emit governance log entries and respect human-oversight gates.
12. `docker compose up` builds and runs the affected services.
```
