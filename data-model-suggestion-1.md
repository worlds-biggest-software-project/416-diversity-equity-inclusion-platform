# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Diversity, Equity & Inclusion Platform (Candidate #416)
> Approach: Traditional normalized relational schema with Row-Level Security

---

## Summary

A fully normalized relational data model implemented in PostgreSQL, using strict third-normal-form (3NF) design for transactional data and a star schema analytical layer for reporting. Multi-tenant data isolation is enforced at the database level using PostgreSQL Row-Level Security (RLS) policies, ensuring that even application-layer bugs cannot leak demographic data across organisations. This is the most conventional and well-understood approach, offering strong ACID guarantees, mature tooling, and straightforward compliance with GDPR Article 9 requirements.

---

## Key Entities and Relationships

### Core Domain Model

```
Organisation (tenant)
 |
 +-- Department
 |    +-- Team
 |
 +-- Employee
 |    +-- EmployeeDemographic (1:1, encrypted/pseudonymised)
 |    +-- EmploymentHistory (1:N, temporal)
 |    +-- Compensation (1:N, temporal)
 |    +-- PerformanceReview (1:N)
 |
 +-- PayEquityAnalysis
 |    +-- PayEquityRun (point-in-time snapshot)
 |    +-- PayEquityResult (per-employee residuals)
 |
 +-- Survey
 |    +-- SurveyQuestion
 |    +-- SurveyResponse (anonymised)
 |    +-- SurveyResponseDemographic (detached, k-anonymity enforced)
 |
 +-- Programme
 |    +-- MentorshipPair
 |    +-- ERG
 |    +-- ERGMembership
 |    +-- ProgrammeParticipation
 |
 +-- Goal
 |    +-- GoalMilestone
 |    +-- GoalProgress (time-series)
 |
 +-- Report
 |    +-- ReportTemplate (EEO-1, CSRD/ESRS S1, GRI 405, etc.)
 |    +-- ReportRun (generated snapshots)
```

### Example Schema Snippets

**Tenant and Organisation Structure**

```sql
CREATE TABLE organisation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    country_code    CHAR(2) NOT NULL,
    data_residency  TEXT NOT NULL CHECK (data_residency IN ('eu', 'us', 'uk', 'apac')),
    gdpr_legal_basis TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE department (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    name            TEXT NOT NULL,
    parent_id       UUID REFERENCES department(id),
    level           INT NOT NULL DEFAULT 0
);
-- RLS policy
ALTER TABLE department ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON department
    USING (organisation_id = current_setting('app.current_org_id')::UUID);
```

**Employee and Demographics (Separated for Privacy)**

```sql
CREATE TABLE employee (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    employee_ref    TEXT NOT NULL,  -- external HRIS identifier
    department_id   UUID REFERENCES department(id),
    job_title       TEXT,
    job_level       INT,
    hire_date       DATE NOT NULL,
    termination_date DATE,
    location_country CHAR(2),
    location_city   TEXT,
    status          TEXT NOT NULL CHECK (status IN ('active', 'terminated', 'leave')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Demographic data stored separately with encryption
-- Access requires elevated RBAC role
CREATE TABLE employee_demographic (
    employee_id     UUID PRIMARY KEY REFERENCES employee(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    -- All demographic fields encrypted at rest via pgcrypto or application-level encryption
    gender          TEXT,  -- encrypted
    ethnicity       TEXT,  -- encrypted
    age_band        TEXT,  -- derived, not raw DOB
    disability_status TEXT, -- encrypted
    veteran_status  TEXT,  -- encrypted
    sexual_orientation TEXT, -- encrypted
    consent_date    TIMESTAMPTZ,
    consent_version TEXT,
    legal_basis     TEXT NOT NULL,  -- 'explicit_consent', 'employment_law', 'public_interest'
    data_source     TEXT NOT NULL CHECK (data_source IN ('self_reported', 'hris_import', 'inferred')),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
-- Separate RLS + role-based access
ALTER TABLE employee_demographic ENABLE ROW LEVEL SECURITY;
CREATE POLICY demographic_access ON employee_demographic
    USING (
        organisation_id = current_setting('app.current_org_id')::UUID
        AND current_setting('app.role') IN ('dei_analyst', 'hr_admin', 'system')
    );
```

**Compensation (Temporal)**

```sql
CREATE TABLE compensation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    effective_date  DATE NOT NULL,
    base_salary     NUMERIC(12,2) NOT NULL,
    currency        CHAR(3) NOT NULL,
    bonus_target    NUMERIC(12,2),
    equity_value    NUMERIC(12,2),
    total_comp      NUMERIC(12,2) GENERATED ALWAYS AS (base_salary + COALESCE(bonus_target, 0) + COALESCE(equity_value, 0)) STORED,
    pay_decision_type TEXT CHECK (pay_decision_type IN ('hire', 'promotion', 'merit', 'adjustment', 'market')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_compensation_employee_date ON compensation(employee_id, effective_date DESC);
```

**Pay Equity Analysis**

```sql
CREATE TABLE pay_equity_run (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    run_date        TIMESTAMPTZ NOT NULL DEFAULT now(),
    model_type      TEXT NOT NULL DEFAULT 'multivariate_regression',
    control_variables TEXT[] NOT NULL DEFAULT ARRAY['job_level', 'tenure', 'geography', 'department'],
    protected_characteristics TEXT[] NOT NULL,
    status          TEXT NOT NULL CHECK (status IN ('pending', 'running', 'completed', 'failed')),
    summary_json    JSONB  -- aggregated results only, no individual PII
);

CREATE TABLE pay_equity_result (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    run_id          UUID NOT NULL REFERENCES pay_equity_run(id),
    employee_id     UUID NOT NULL REFERENCES employee(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    predicted_comp  NUMERIC(12,2),
    actual_comp     NUMERIC(12,2),
    residual        NUMERIC(12,2),
    flag            TEXT CHECK (flag IN ('within_range', 'underpaid', 'overpaid', 'review'))
);
```

**Inclusion Surveys (Anonymised)**

```sql
CREATE TABLE survey (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    title           TEXT NOT NULL,
    survey_type     TEXT NOT NULL CHECK (survey_type IN ('pulse', 'annual', 'onboarding', 'exit')),
    instrument_version TEXT NOT NULL,
    anonymity_threshold INT NOT NULL DEFAULT 5,  -- minimum respondents per group
    status          TEXT NOT NULL CHECK (status IN ('draft', 'active', 'closed', 'archived')),
    opens_at        TIMESTAMPTZ,
    closes_at       TIMESTAMPTZ
);

-- Responses use a pseudonymous ID, NOT employee_id
CREATE TABLE survey_response (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id       UUID NOT NULL REFERENCES survey(id),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    respondent_pseudo_id UUID NOT NULL,  -- NOT linked to employee table
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE survey_answer (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    response_id     UUID NOT NULL REFERENCES survey_response(id),
    question_id     UUID NOT NULL REFERENCES survey_question(id),
    score           INT,           -- Likert scale (1-5)
    text_response   TEXT,          -- open-ended
    dimension       TEXT           -- 'belonging', 'psychological_safety', 'fair_treatment', 'voice'
);
```

**Reporting and Compliance**

```sql
CREATE TABLE report_template (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    framework       TEXT NOT NULL CHECK (framework IN ('eeo1', 'csrd_esrs_s1', 'gri_405', 'bloomberg_gei', 'pay_transparency_directive', 'custom')),
    version         TEXT NOT NULL,
    schema_json     JSONB NOT NULL,  -- expected data structure
    active          BOOLEAN NOT NULL DEFAULT true
);

CREATE TABLE report_run (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisation(id),
    template_id     UUID NOT NULL REFERENCES report_template(id),
    reporting_period_start DATE NOT NULL,
    reporting_period_end   DATE NOT NULL,
    generated_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    data_snapshot   JSONB NOT NULL,  -- aggregated, anonymised data
    status          TEXT NOT NULL CHECK (status IN ('draft', 'review', 'approved', 'submitted'))
);
```

### Analytical Layer (Star Schema Views)

```sql
-- Dimension: Employee (aggregated, no PII in analytical layer)
CREATE MATERIALIZED VIEW dim_employee AS
SELECT
    e.id,
    e.organisation_id,
    e.department_id,
    d.name AS department_name,
    e.job_level,
    e.location_country,
    e.status,
    ed.gender,
    ed.ethnicity,
    ed.age_band,
    ed.disability_status,
    ed.veteran_status,
    DATE_PART('year', AGE(CURRENT_DATE, e.hire_date)) AS tenure_years
FROM employee e
JOIN employee_demographic ed ON ed.employee_id = e.id
JOIN department d ON d.id = e.department_id;

-- Fact: Workforce Events
CREATE MATERIALIZED VIEW fact_workforce_event AS
SELECT
    gen_random_uuid() AS id,
    eh.employee_id,
    eh.organisation_id,
    eh.event_type,     -- 'hire', 'promotion', 'transfer', 'termination'
    eh.event_date,
    eh.from_department_id,
    eh.to_department_id,
    eh.from_level,
    eh.to_level
FROM employment_history eh;
```

---

## Pros

- **Mature and well-understood.** Relational databases are the most widely adopted data storage technology. PostgreSQL has extensive documentation, a vast ecosystem of tools (ORMs, migration frameworks, monitoring), and a deep talent pool.
- **Strong ACID guarantees.** Transactional integrity is critical when processing sensitive demographic data and compensation records. Relational databases provide this natively.
- **Row-Level Security for multi-tenancy.** PostgreSQL RLS enforces tenant isolation at the database engine level, providing defence-in-depth that does not depend on application code correctness. This is especially valuable for GDPR Article 9 special category data.
- **Separation of demographics from identity.** The split between `employee` and `employee_demographic` tables enables different access control policies, encryption strategies, and audit logging for sensitive vs. non-sensitive data.
- **Compliance-friendly audit trails.** Trigger-based audit logging (via `pg_audit` or custom triggers) produces a complete, tamper-evident record of every access to and modification of demographic data.
- **Star schema analytical layer.** Materialised views provide fast analytical queries for dashboards without denormalising the transactional schema.
- **Regulatory reporting alignment.** The structured, predictable schema maps cleanly to CSRD/ESRS S1, EEO-1, and GRI 405 reporting requirements.

## Cons

- **Schema rigidity.** Adding new demographic dimensions (e.g., neurodiversity, socioeconomic background) requires schema migrations. In a domain where demographic categories are evolving, this can create friction.
- **Complex joins for intersectional analysis.** Queries that cross-tabulate multiple demographic dimensions with workforce events, compensation, and survey data require multi-table joins that can be expensive at scale.
- **Analytical performance at scale.** Materialised views help, but very large organisations (100,000+ employees with years of historical data) may find that a pure PostgreSQL analytical layer becomes a bottleneck without a dedicated OLAP solution.
- **Survey anonymisation complexity.** Maintaining the firewall between `survey_response` (pseudonymised) and `employee` (identified) requires careful application logic. A bug that leaks the mapping between `respondent_pseudo_id` and `employee_id` would compromise survey anonymity.
- **Multi-jurisdiction demographic taxonomies.** Different countries use different categories for ethnicity, gender, and disability. Encoding these as enum values or lookup tables requires careful per-jurisdiction configuration.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| Primary database | PostgreSQL 16+ (native RLS, JSONB, generated columns) |
| Encryption at rest | Application-level encryption (pgcrypto or Vault) for demographic fields |
| Audit logging | pgAudit extension + custom trigger-based audit tables |
| Migration tooling | Flyway or Liquibase for versioned schema migrations |
| ORM layer | Prisma, Drizzle, or SQLAlchemy with explicit RLS session management |
| Analytical layer | Materialised views for small-to-mid deployments; Apache Superset or Metabase for dashboarding |
| HRIS integration | Merge or Finch unified HRIS API, writing to staging tables with ETL pipeline |
| Statistical engine | Python (statsmodels, scikit-learn) for pay equity regression, invoked via background workers |

---

## Migration and Scaling Considerations

- **Horizontal read scaling:** PostgreSQL logical replication to read replicas for analytical workloads. Materialised views can be refreshed on replicas to avoid impacting transactional performance.
- **Data partitioning:** Partition large tables (compensation history, employment history, survey responses) by `organisation_id` using PostgreSQL declarative partitioning for improved query performance in multi-tenant scenarios.
- **OLAP offload:** For organisations exceeding 50,000 employees, consider offloading analytical queries to a columnar store (ClickHouse, DuckDB, or BigQuery) via CDC (Change Data Capture) with Debezium. The normalised transactional schema remains the system of record.
- **Right to erasure (GDPR Article 17):** Deletion of an employee's demographic data is straightforward in a normalised schema -- delete from `employee_demographic` and cascade or anonymise related records. The relational model's referential integrity makes it clear which records must be addressed.
- **Multi-region deployment:** For data residency requirements (GDPR), deploy separate PostgreSQL instances per data residency region with `organisation.data_residency` routing at the application layer. Consider CockroachDB or Citus for globally distributed PostgreSQL if single-instance-per-region becomes operationally burdensome.
- **Schema evolution:** Use versioned migrations with backward-compatible changes (add columns as nullable, never rename in place). Demographic taxonomy changes should use lookup tables rather than CHECK constraints to avoid migration-heavy category updates.
