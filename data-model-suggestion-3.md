# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Diversity, Equity & Inclusion Platform (Candidate #416)
> Approach: PostgreSQL relational core with JSONB columns for flexible and jurisdiction-specific data

---

## Summary

A pragmatic hybrid architecture that uses PostgreSQL as a single database engine, combining normalised relational tables for well-defined core entities (employees, compensation, surveys, programmes) with JSONB columns for inherently variable data -- demographic taxonomies that differ by jurisdiction, survey instruments that evolve over time, HRIS connector payloads that vary by source system, and compliance report schemas that differ by regulatory framework. This approach provides the referential integrity and ACID guarantees of a relational database while accommodating the schema variability that is intrinsic to a multi-jurisdictional DEI platform without requiring a separate document database.

---

## Key Entities and Relationships

### Design Principles

The hybrid model applies a clear rule: **use relational columns for data that is structurally stable and queried frequently; use JSONB for data that varies by jurisdiction, source system, or reporting framework.** Specifically:

- **Relational:** Employee identity, organisational structure, compensation amounts, survey structure, programme structure, goal definitions
- **JSONB:** Demographic attributes (vary by country), HRIS raw payloads (vary by source), survey question configurations, compliance report data (vary by framework), AI analysis outputs, custom metadata

### Example Schema Snippets

**Organisation and Multi-Jurisdiction Configuration**

```sql
CREATE TABLE organisation (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name              TEXT NOT NULL,
    primary_country   CHAR(2) NOT NULL,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- JSONB: jurisdiction-specific configuration
    demographic_config JSONB NOT NULL DEFAULT '{}'::JSONB,
    /*
    Example:
    {
        "jurisdictions": {
            "US": {
                "ethnicity_categories": ["Hispanic/Latino", "White", "Black/African American",
                    "Asian", "Native Hawaiian/Pacific Islander", "American Indian/Alaska Native",
                    "Two or More Races"],
                "gender_categories": ["Male", "Female", "Non-Binary", "Prefer not to say"],
                "additional_fields": ["veteran_status"],
                "legal_basis": "eeo1_voluntary_self_id",
                "anonymity_threshold": 5
            },
            "UK": {
                "ethnicity_categories": ["White", "Mixed/Multiple ethnic groups",
                    "Asian/Asian British", "Black/Black British", "Other ethnic group"],
                "gender_categories": ["Male", "Female", "Non-binary", "Prefer not to say"],
                "additional_fields": ["disability_status", "sexual_orientation", "religion"],
                "legal_basis": "uk_gdpr_schedule1_condition8",
                "anonymity_threshold": 10
            },
            "DE": {
                "ethnicity_categories": [],  -- not collected in Germany
                "gender_categories": ["Maennlich", "Weiblich", "Divers", "Keine Angabe"],
                "additional_fields": ["disability_degree"],
                "legal_basis": "gdpr_art9_employment_law",
                "anonymity_threshold": 15
            }
        }
    }
    */

    privacy_config    JSONB NOT NULL DEFAULT '{}'::JSONB,
    /*
    {
        "consent_required": true,
        "data_retention_months": 36,
        "dpia_completed": true,
        "dpia_date": "2026-03-15",
        "encryption_policy": "field_level_aes256",
        "data_residency_region": "eu-west-1"
    }
    */

    reporting_config  JSONB NOT NULL DEFAULT '{}'::JSONB
    /*
    {
        "enabled_frameworks": ["csrd_esrs_s1", "gri_405", "pay_transparency_directive"],
        "reporting_periods": {"csrd_esrs_s1": "annual", "pay_transparency_directive": "annual"},
        "auto_generate": true
    }
    */
);
```

**Employee with JSONB Demographics**

```sql
CREATE TABLE employee (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id   UUID NOT NULL REFERENCES organisation(id),
    employee_ref      TEXT NOT NULL,        -- HRIS external identifier
    department_id     UUID REFERENCES department(id),
    job_title         TEXT,
    job_level         INT,
    hire_date         DATE NOT NULL,
    termination_date  DATE,
    status            TEXT NOT NULL CHECK (status IN ('active', 'terminated', 'leave')),
    location_country  CHAR(2),
    location_city     TEXT,

    -- JSONB: demographic data (varies by jurisdiction)
    demographics      JSONB DEFAULT '{}'::JSONB,
    /*
    US employee example:
    {
        "gender": "Female",
        "ethnicity": "Asian",
        "veteran_status": "Non-Veteran",
        "disability_status": "No",
        "consent": {"date": "2026-01-15", "version": "2.1", "method": "self_service_portal"},
        "source": "self_reported",
        "collected_at": "2026-01-15T10:30:00Z"
    }

    DE employee example:
    {
        "gender": "Weiblich",
        "disability_degree": 0,
        "consent": {"date": "2026-02-01", "version": "3.0", "method": "onboarding_form"},
        "source": "hris_import",
        "collected_at": "2026-02-01T08:00:00Z"
    }
    */

    -- JSONB: raw HRIS payload (preserves source system data)
    hris_raw          JSONB DEFAULT '{}'::JSONB,

    -- JSONB: custom attributes per organisation
    custom_attributes JSONB DEFAULT '{}'::JSONB,

    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),

    UNIQUE (organisation_id, employee_ref)
);

-- RLS for multi-tenant isolation
ALTER TABLE employee ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON employee
    USING (organisation_id = current_setting('app.current_org_id')::UUID);

-- GIN index on demographics JSONB for analytical queries
CREATE INDEX idx_employee_demographics ON employee USING GIN (demographics jsonb_path_ops);

-- Partial index for active employees (most common query filter)
CREATE INDEX idx_employee_active ON employee (organisation_id, department_id)
    WHERE status = 'active';

-- Expression indexes on commonly queried demographic fields
CREATE INDEX idx_employee_gender ON employee ((demographics->>'gender'))
    WHERE demographics->>'gender' IS NOT NULL;
CREATE INDEX idx_employee_ethnicity ON employee ((demographics->>'ethnicity'))
    WHERE demographics->>'ethnicity' IS NOT NULL;
```

**Compensation (Relational -- Structurally Stable)**

```sql
CREATE TABLE compensation (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id       UUID NOT NULL REFERENCES employee(id),
    organisation_id   UUID NOT NULL REFERENCES organisation(id),
    effective_date    DATE NOT NULL,
    base_salary       NUMERIC(12,2) NOT NULL,
    currency          CHAR(3) NOT NULL,
    bonus_target      NUMERIC(12,2),
    equity_value      NUMERIC(12,2),
    total_comp        NUMERIC(12,2) GENERATED ALWAYS AS
        (base_salary + COALESCE(bonus_target, 0) + COALESCE(equity_value, 0)) STORED,
    decision_type     TEXT CHECK (decision_type IN ('hire', 'promotion', 'merit', 'adjustment', 'market')),

    -- JSONB: additional comp components that vary by country/org
    additional_components JSONB DEFAULT '{}'::JSONB,
    /*
    {
        "housing_allowance": 2000,
        "car_allowance": 500,
        "pension_contribution_pct": 5.0,
        "overtime_eligible": false
    }
    */

    created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Pay Equity Analysis**

```sql
CREATE TABLE pay_equity_run (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id   UUID NOT NULL REFERENCES organisation(id),
    run_date          TIMESTAMPTZ NOT NULL DEFAULT now(),
    status            TEXT NOT NULL CHECK (status IN ('pending', 'running', 'completed', 'failed')),

    -- JSONB: model configuration (varies by analysis type)
    model_config      JSONB NOT NULL,
    /*
    {
        "model_type": "multivariate_ols_regression",
        "dependent_variable": "total_comp",
        "control_variables": ["job_level", "tenure_years", "location_country", "department"],
        "protected_characteristics": ["gender", "ethnicity"],
        "interaction_terms": [["gender", "ethnicity"]],
        "filters": {"status": "active", "min_tenure_months": 6},
        "confidence_level": 0.95
    }
    */

    -- JSONB: aggregated results (no individual PII)
    results_summary   JSONB,
    /*
    {
        "overall_gap": {"gender": {"female_vs_male": -0.032, "p_value": 0.0012}},
        "by_department": [
            {"dept": "Engineering", "gap": -0.045, "p_value": 0.003, "n": 245},
            {"dept": "Sales", "gap": -0.018, "p_value": 0.12, "n": 89}
        ],
        "intersectional": [
            {"group": "Black Female", "gap": -0.067, "p_value": 0.0001, "n": 34}
        ],
        "model_r_squared": 0.82,
        "alerts_generated": 3
    }
    */

    created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Surveys with Flexible Instruments**

```sql
CREATE TABLE survey (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id   UUID NOT NULL REFERENCES organisation(id),
    title             TEXT NOT NULL,
    survey_type       TEXT NOT NULL CHECK (survey_type IN ('pulse', 'annual', 'onboarding', 'exit', 'custom')),
    anonymity_threshold INT NOT NULL DEFAULT 5,
    status            TEXT NOT NULL CHECK (status IN ('draft', 'active', 'closed', 'archived')),
    opens_at          TIMESTAMPTZ,
    closes_at         TIMESTAMPTZ,

    -- JSONB: survey instrument definition (varies per survey)
    instrument        JSONB NOT NULL,
    /*
    {
        "version": "2.0",
        "dimensions": [
            {
                "id": "belonging",
                "label": {"en": "Belonging", "de": "Zugehoerigkeit", "fr": "Appartenance"},
                "questions": [
                    {
                        "id": "q1",
                        "text": {"en": "I feel I belong at this organisation"},
                        "type": "likert_5",
                        "anchors": {"en": ["Strongly Disagree", "Strongly Agree"]}
                    },
                    {
                        "id": "q2",
                        "text": {"en": "I can be my authentic self at work"},
                        "type": "likert_5"
                    }
                ]
            },
            {
                "id": "psychological_safety",
                "label": {"en": "Psychological Safety"},
                "questions": [...]
            }
        ],
        "open_ended": [
            {"id": "open1", "text": {"en": "What could we do to make this a more inclusive workplace?"}}
        ],
        "demographic_questions": ["gender", "ethnicity", "department", "tenure_band"]
    }
    */

    created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Survey responses use pseudonymous ID
CREATE TABLE survey_response (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id         UUID NOT NULL REFERENCES survey(id),
    organisation_id   UUID NOT NULL REFERENCES organisation(id),
    respondent_pseudo_id UUID NOT NULL,  -- NOT linked to employee table
    submitted_at      TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- JSONB: all answers in a single document
    answers           JSONB NOT NULL,
    /*
    {
        "scores": {"q1": 4, "q2": 5, "q3": 3, ...},
        "open_text": {"open1": "More flexible working arrangements would help..."},
        "demographics": {"gender": "Female", "tenure_band": "2-5 years", "department": "Engineering"}
    }
    */

    -- JSONB: AI-generated analysis (added post-submission)
    ai_analysis       JSONB DEFAULT '{}'::JSONB
    /*
    {
        "sentiment": "positive",
        "themes": ["flexibility", "remote_work", "career_development"],
        "inclusion_signals": {"belonging": "high", "voice": "moderate"}
    }
    */
);
```

**Compliance Reporting (Framework-Specific JSONB)**

```sql
CREATE TABLE compliance_report (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id   UUID NOT NULL REFERENCES organisation(id),
    framework         TEXT NOT NULL,
    reporting_period_start DATE NOT NULL,
    reporting_period_end   DATE NOT NULL,
    status            TEXT NOT NULL CHECK (status IN ('draft', 'review', 'approved', 'submitted')),

    -- JSONB: report data structured per framework requirements
    report_data       JSONB NOT NULL,
    /*
    EEO-1 example:
    {
        "establishment_info": {"name": "HQ", "naics": "541511"},
        "categories": [
            {
                "job_category": "Executive/Senior-Level Officials",
                "data": {
                    "Hispanic_Latino": {"male": 2, "female": 1},
                    "White": {"male": 15, "female": 8},
                    "Black_African_American": {"male": 3, "female": 2},
                    ...
                }
            }
        ]
    }

    CSRD/ESRS S1 example:
    {
        "s1_9_diversity": {
            "gender_top_management": {"male": 65, "female": 35},
            "age_distribution": {"under_30": 22, "30_50": 58, "over_50": 20}
        },
        "s1_12_disability": {"percentage": 4.2},
        "s1_16_pay_gap": {"gender_pay_gap_mean": 8.3, "gender_pay_gap_median": 5.1}
    }
    */

    generated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    approved_by       UUID,
    approved_at       TIMESTAMPTZ,

    UNIQUE (organisation_id, framework, reporting_period_start, reporting_period_end)
);
```

**Programme Management**

```sql
CREATE TABLE programme (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id   UUID NOT NULL REFERENCES organisation(id),
    name              TEXT NOT NULL,
    programme_type    TEXT NOT NULL CHECK (programme_type IN ('mentorship', 'erg', 'sponsorship', 'training', 'custom')),
    status            TEXT NOT NULL CHECK (status IN ('planning', 'active', 'paused', 'completed')),
    start_date        DATE,
    end_date          DATE,

    -- JSONB: programme-type-specific configuration
    config            JSONB NOT NULL DEFAULT '{}'::JSONB,
    /*
    Mentorship example:
    {
        "matching_criteria": ["skills", "career_goals", "diversity_dimensions", "department"],
        "matching_algorithm": "weighted_compatibility",
        "session_frequency": "biweekly",
        "duration_months": 6,
        "max_pairs": 50,
        "cross_department": true
    }

    ERG example:
    {
        "affinity_group": "Women in Tech",
        "open_to_allies": true,
        "budget_annual": 15000,
        "executive_sponsor_id": "uuid-here"
    }
    */

    -- JSONB: programme outcomes and analytics
    outcomes          JSONB DEFAULT '{}'::JSONB,
    /*
    {
        "participants": 87,
        "completion_rate": 0.82,
        "nps_score": 72,
        "promotion_rate_participants": 0.18,
        "promotion_rate_non_participants": 0.11,
        "retention_rate_participants": 0.94,
        "retention_rate_non_participants": 0.86
    }
    */

    created_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE mentorship_pair (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    programme_id      UUID NOT NULL REFERENCES programme(id),
    mentor_id         UUID NOT NULL REFERENCES employee(id),
    mentee_id         UUID NOT NULL REFERENCES employee(id),
    organisation_id   UUID NOT NULL REFERENCES organisation(id),
    matched_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    status            TEXT NOT NULL CHECK (status IN ('active', 'completed', 'cancelled')),
    compatibility_score NUMERIC(3,2),

    -- JSONB: matching rationale
    match_rationale   JSONB DEFAULT '{}'::JSONB
    /*
    {
        "shared_skills": ["data_science", "python"],
        "complementary_goals": ["leadership", "technical_depth"],
        "diversity_pairing": {"mentor_gender": "Male", "mentee_gender": "Female", "cross_department": true}
    }
    */
);
```

**Audit Log**

```sql
CREATE TABLE audit_log (
    id                BIGSERIAL PRIMARY KEY,
    organisation_id   UUID NOT NULL,
    actor_id          UUID,
    actor_role        TEXT NOT NULL,
    action            TEXT NOT NULL,  -- 'read', 'create', 'update', 'delete', 'export'
    resource_type     TEXT NOT NULL,  -- 'employee_demographic', 'compensation', 'survey_response', etc.
    resource_id       UUID,
    timestamp         TIMESTAMPTZ NOT NULL DEFAULT now(),

    -- JSONB: action-specific details
    details           JSONB DEFAULT '{}'::JSONB
    /*
    {
        "fields_accessed": ["gender", "ethnicity"],
        "query_context": "pay_equity_analysis",
        "ip_address": "10.0.1.42",
        "justification": "Q1 2026 pay equity audit"
    }
    */
);

-- Append-only: prevent modification
CREATE OR REPLACE FUNCTION prevent_audit_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Audit log entries cannot be modified or deleted';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_immutable
    BEFORE UPDATE OR DELETE ON audit_log
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_mutation();
```

---

## Pros

- **Single database engine simplicity.** PostgreSQL handles both relational and document storage, eliminating the operational burden of running a separate document database (MongoDB, DynamoDB). One backup strategy, one monitoring stack, one set of connection pools.
- **Schema flexibility where it matters.** Demographic categories vary by jurisdiction (US EEO-1 categories vs. UK Census categories vs. Germany where ethnicity is not collected). JSONB columns accommodate these differences without schema migrations or per-country tables.
- **ACID guarantees on flexible data.** Unlike a separate document database, JSONB columns within PostgreSQL participate in the same transaction as relational columns. A compensation update and its associated demographic cross-reference are atomically consistent.
- **GIN indexing on JSONB.** PostgreSQL GIN indexes enable fast queries on JSONB fields (`WHERE demographics->>'gender' = 'Female'`), and expression indexes can target the most commonly queried demographic dimensions specifically.
- **Multi-lingual survey instruments.** Storing survey instruments as JSONB with nested language keys (`{"en": "...", "de": "...", "fr": "..."}`) naturally supports multilingual deployments without separate localisation tables.
- **HRIS payload preservation.** Storing raw HRIS payloads in `hris_raw` JSONB alongside normalised relational fields enables debugging integration issues and re-processing data without re-fetching from the source system.
- **Compliance report flexibility.** Each regulatory framework (EEO-1, CSRD/ESRS S1, GRI 405, Bloomberg GEI) has a different data structure. JSONB `report_data` columns accommodate framework-specific schemas without a separate table per framework.
- **Gradual schema evolution.** New demographic dimensions can be added to the JSONB schema without database migrations. Commonly queried fields can be promoted to dedicated relational columns later as usage patterns stabilise.
- **Row-Level Security compatible.** RLS policies work on tables containing JSONB columns identically to pure relational tables, maintaining multi-tenant data isolation.

## Cons

- **Inconsistent data risk.** Without schema enforcement, JSONB columns can contain inconsistent or invalid data. Two employees in the same US office might have `demographics.ethnicity` values that use different category names. CHECK constraints and application-level validation mitigate this but do not eliminate it.
- **Query complexity.** Querying across relational and JSONB columns requires mixing SQL operators with JSONB operators (`->>`, `@>`, `jsonb_path_query`), which can be harder to read and maintain than pure relational queries.
- **Index management.** GIN indexes on JSONB are powerful but more expensive to maintain than B-tree indexes on relational columns. Expression indexes on specific JSONB paths improve query performance but require manual creation as query patterns emerge.
- **Reporting joins.** Analytical queries that join compensation (relational), demographics (JSONB), and survey results (JSONB) across large employee populations may be slower than equivalent queries on a fully denormalised relational schema because the JSONB extraction happens at query time.
- **ORM friction.** Most ORMs have limited support for JSONB queries. Complex JSONB queries often require raw SQL, reducing the benefits of using an ORM for type safety and query building.
- **Data validation discipline.** The team must maintain rigorous application-level validation and potentially JSON Schema validation (using `IS JSON` predicates in PostgreSQL 16+ and CHECK constraints) to prevent JSONB columns from becoming a dumping ground for unstructured data.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| Primary database | PostgreSQL 16+ (enhanced JSONB, IS JSON predicates, GIN improvements) |
| JSONB validation | JSON Schema validation at application layer (ajv for TypeScript, jsonschema for Python); CHECK constraints for critical fields |
| ORM | Prisma (good JSONB support), Drizzle (raw SQL escape hatch), or Kysely (type-safe query builder with JSONB support) |
| Migration tooling | Flyway or Liquibase for relational schema; application-level migration scripts for JSONB schema changes |
| Encryption | Application-level AES-256 encryption on `demographics` JSONB field before storage; HashiCorp Vault for key management |
| Search | PostgreSQL full-text search on survey open-text responses; upgrade to Elasticsearch/OpenSearch if volume exceeds 1M responses |
| Dashboarding | Apache Superset or Metabase (both have good PostgreSQL JSONB support) |
| HRIS integration | Merge or Finch unified HRIS API; store normalised data in relational columns + raw payload in `hris_raw` JSONB |
| Statistical engine | Python (statsmodels) consuming data via SQLAlchemy with JSONB extraction |

---

## Migration and Scaling Considerations

- **JSONB to relational promotion.** If a demographic dimension (e.g., `gender`) is queried in 90%+ of analytical queries, promote it from JSONB to a dedicated relational column with a backfill migration. The expression index strategy provides a stepping stone: index first, promote to column later if needed.
- **Read replicas for analytics.** Use PostgreSQL streaming replication to route analytical queries (dashboards, pay equity models, compliance reports) to read replicas, keeping the primary database responsive for transactional writes (HRIS sync, survey submission).
- **JSONB field-level encryption.** Encrypt the entire `demographics` JSONB value rather than individual fields. This simplifies key management (one key per employee, not one per field) while ensuring that decryption is only performed by authorised application code paths.
- **GDPR right to erasure.** Set `demographics` to `'{}'::JSONB` and `hris_raw` to `'{}'::JSONB` for the target employee. Relational fields (department, job level, hire date) can be retained or anonymised based on data retention policy. The JSONB approach makes erasure simpler than managing multiple relational tables.
- **Multi-region data residency.** Deploy separate PostgreSQL instances per data residency region. The `organisation.privacy_config.data_residency_region` JSONB field drives application-level routing. For simpler deployments, use a single Citus-distributed PostgreSQL cluster with tenant-based sharding.
- **Performance at scale.** For organisations with 100,000+ employees and years of historical data, consider materialised views that pre-extract JSONB fields into relational columns for the analytical layer. Refresh materialised views on a schedule (hourly or daily) rather than on every write.
- **Schema versioning for JSONB.** Include a `_schema_version` key in JSONB objects to enable backward-compatible evolution. Application code can handle multiple schema versions, and batch migration scripts can upgrade old records in the background.
