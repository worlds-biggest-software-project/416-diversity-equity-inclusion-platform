# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Diversity, Equity & Inclusion Platform (Candidate #416)
> Approach: Event Sourcing with Command Query Responsibility Segregation

---

## Summary

An event-sourced architecture where every change to workforce data is captured as an immutable event in an append-only event store, with separate read models (projections) materialised for different query needs -- dashboards, pay equity analysis, compliance reports, and survey analytics. CQRS separates the write path (commands that produce events) from the read path (projections optimised for specific query patterns). This approach provides a complete, tamper-evident audit trail of every demographic change, compensation decision, and survey submission -- a significant advantage for regulatory compliance and pay equity litigation defensibility.

---

## Key Entities and Relationships

### Domain Aggregates and Event Streams

```
Employee Aggregate (stream: employee-{id})
  Events: EmployeeHired, EmployeeTransferred, EmployeePromoted,
          CompensationChanged, EmployeeTerminated, EmployeeReinstated

Demographic Aggregate (stream: demographic-{employee_id})
  Events: DemographicSelfReported, DemographicUpdated, DemographicWithdrawn,
          ConsentGranted, ConsentRevoked
  (Encrypted payload; separate stream for access control)

PayEquity Aggregate (stream: payequity-{org_id}-{run_id})
  Events: PayEquityAnalysisRequested, PayEquityModelExecuted,
          PayEquityResultsPublished, PayEquityAlertTriggered

Survey Aggregate (stream: survey-{survey_id})
  Events: SurveyCreated, SurveyPublished, SurveyClosed,
          ResponseSubmitted (anonymised), ThemeAnalysisCompleted

Programme Aggregate (stream: programme-{programme_id})
  Events: ProgrammeCreated, MentorshipPairMatched, ERGMemberJoined,
          ERGEventScheduled, ParticipationRecorded

Goal Aggregate (stream: goal-{goal_id})
  Events: GoalSet, MilestoneAchieved, GoalProgressRecorded, GoalClosed

Report Aggregate (stream: report-{report_id})
  Events: ReportRequested, ReportGenerated, ReportApproved, ReportSubmitted
```

### Event Store Schema

```sql
-- Core event store table (append-only)
CREATE TABLE event_store (
    global_position  BIGSERIAL PRIMARY KEY,
    stream_id        TEXT NOT NULL,
    stream_position  INT NOT NULL,
    event_type       TEXT NOT NULL,
    event_data       BYTEA NOT NULL,        -- encrypted JSON payload
    metadata         JSONB NOT NULL,         -- correlation_id, causation_id, actor, timestamp
    organisation_id  UUID NOT NULL,
    occurred_at      TIMESTAMPTZ NOT NULL,
    recorded_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    encryption_key_id TEXT,                  -- reference to key in Vault for crypto-shredding
    UNIQUE (stream_id, stream_position)
);

CREATE INDEX idx_event_store_stream ON event_store(stream_id, stream_position);
CREATE INDEX idx_event_store_type ON event_store(event_type, occurred_at);
CREATE INDEX idx_event_store_org ON event_store(organisation_id, occurred_at);

-- Immutability enforced via trigger
CREATE OR REPLACE FUNCTION prevent_event_mutation()
RETURNS TRIGGER AS $$
BEGIN
    RAISE EXCEPTION 'Events are immutable and cannot be modified or deleted';
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER no_update_delete
    BEFORE UPDATE OR DELETE ON event_store
    FOR EACH ROW EXECUTE FUNCTION prevent_event_mutation();
```

### Example Event Definitions

```typescript
// Employee lifecycle events
interface EmployeeHired {
    type: 'EmployeeHired';
    data: {
        employeeId: string;
        organisationId: string;
        departmentId: string;
        jobTitle: string;
        jobLevel: number;
        locationCountry: string;
        hireDate: string;        // ISO 8601
        hrisSourceRef: string;
    };
    metadata: {
        correlationId: string;
        causationId: string;     // e.g., HRIS sync event that triggered this
        actor: string;           // system or user who recorded
        timestamp: string;
    };
}

interface CompensationChanged {
    type: 'CompensationChanged';
    data: {
        employeeId: string;
        effectiveDate: string;
        baseSalary: number;
        currency: string;
        bonusTarget: number | null;
        equityValue: number | null;
        decisionType: 'hire' | 'promotion' | 'merit' | 'adjustment' | 'market';
        previousBaseSalary: number | null;
        changePercentage: number | null;
    };
}

// Demographic events (encrypted payload)
interface DemographicSelfReported {
    type: 'DemographicSelfReported';
    data: {
        employeeId: string;
        encryptedPayload: string;  // AES-256 encrypted JSON containing actual demographics
        encryptionKeyId: string;   // Reference to per-employee key in Vault
        consentVersion: string;
        legalBasis: 'explicit_consent' | 'employment_law' | 'public_interest';
        dataFields: string[];      // list of fields provided, without values (for audit)
    };
}

// Pay equity events
interface PayEquityAlertTriggered {
    type: 'PayEquityAlertTriggered';
    data: {
        runId: string;
        alertType: 'gap_widened' | 'new_gap_detected' | 'threshold_exceeded';
        protectedCharacteristic: string;
        departmentId: string;
        gapPercentage: number;
        statisticalSignificance: number;
        affectedEmployeeCount: number;  // count only, no PII
    };
}
```

### Read Model Projections

```sql
-- Projection: Current Employee State (materialised from event stream)
CREATE TABLE projection_employee_current (
    employee_id     UUID PRIMARY KEY,
    organisation_id UUID NOT NULL,
    department_id   UUID,
    job_title       TEXT,
    job_level       INT,
    location_country CHAR(2),
    status          TEXT,
    hire_date       DATE,
    termination_date DATE,
    current_base_salary NUMERIC(12,2),
    currency        CHAR(3),
    last_event_position BIGINT NOT NULL,  -- watermark for replay
    updated_at      TIMESTAMPTZ NOT NULL
);

-- Projection: Workforce Demographics Dashboard
-- (aggregated, never stores individual-level demographic data)
CREATE TABLE projection_demographics_aggregate (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL,
    snapshot_date   DATE NOT NULL,
    department_id   UUID,
    job_level       INT,
    location_country CHAR(2),
    gender          TEXT,
    ethnicity       TEXT,
    headcount       INT NOT NULL,
    avg_tenure_months NUMERIC(6,1),
    avg_compensation NUMERIC(12,2),
    UNIQUE (organisation_id, snapshot_date, department_id, job_level, location_country, gender, ethnicity)
);

-- Projection: Pay Equity Results (latest run)
CREATE TABLE projection_pay_equity_latest (
    organisation_id UUID NOT NULL,
    run_id          UUID NOT NULL,
    run_date        TIMESTAMPTZ NOT NULL,
    protected_characteristic TEXT NOT NULL,
    department_id   UUID,
    gap_percentage  NUMERIC(5,2),
    statistical_significance NUMERIC(5,4),
    affected_count  INT,
    alert_status    TEXT,
    PRIMARY KEY (organisation_id, run_id, protected_characteristic, department_id)
);

-- Projection: Survey Results (anonymised aggregates)
CREATE TABLE projection_survey_results (
    survey_id       UUID NOT NULL,
    organisation_id UUID NOT NULL,
    dimension       TEXT NOT NULL,  -- belonging, psychological_safety, etc.
    demographic_group TEXT,         -- only shown if >= anonymity_threshold respondents
    avg_score       NUMERIC(3,2),
    response_count  INT NOT NULL,
    themes          TEXT[],         -- AI-extracted themes from open text
    PRIMARY KEY (survey_id, dimension, demographic_group)
);

-- Projection: Compliance Report Data
CREATE TABLE projection_compliance_data (
    organisation_id UUID NOT NULL,
    framework       TEXT NOT NULL,  -- 'eeo1', 'csrd_esrs_s1', 'gri_405', etc.
    reporting_period TEXT NOT NULL,
    data_json       JSONB NOT NULL,  -- pre-computed report data in framework schema
    generated_at    TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (organisation_id, framework, reporting_period)
);
```

### GDPR Compliance: Crypto-Shredding

```
Per-Employee Encryption Architecture:

  Event Store                    Key Management (HashiCorp Vault)
  +---------------------------+  +----------------------------+
  | stream: demographic-emp1  |  | Key: emp1-demo-key         |
  | payload: [AES-encrypted]  |--| Status: active             |
  | key_id: emp1-demo-key     |  +----------------------------+
  +---------------------------+

  On GDPR Article 17 "right to erasure" request:
  1. Destroy encryption key in Vault
  2. Encrypted events become permanently unreadable (crypto-shredded)
  3. Event structure remains intact (stream integrity preserved)
  4. Rebuild projections -- demographic fields become NULL
  5. Aggregated statistics remain valid (individual cannot be re-identified)
```

---

## Pros

- **Complete audit trail.** Every workforce change, compensation decision, and demographic update is recorded as an immutable event. This is invaluable for pay equity litigation, regulatory audits, and CSRD compliance where organisations must demonstrate how their workforce data evolved over time.
- **Time-travel queries.** The event store enables replaying events to any point in time, answering questions like "What was the gender pay gap in the engineering department on 1 January 2025?" without maintaining separate snapshot tables.
- **Crypto-shredding for GDPR compliance.** By encrypting demographic event payloads with per-employee keys stored in a key management system (HashiCorp Vault), GDPR Article 17 "right to erasure" requests are handled by destroying the encryption key. The event stream's structural integrity is preserved while the personal data becomes permanently unreadable.
- **Independent read model optimisation.** Each projection (dashboard, pay equity, compliance reporting) is optimised independently for its specific query patterns. The dashboard projection can be a denormalised aggregate table; the compliance projection can match the exact schema required by CSRD/ESRS S1 or EEO-1.
- **Decoupled processing.** Event-driven architecture naturally supports asynchronous processing: pay equity models, AI root-cause analysis, and ESG report generation subscribe to relevant events and process independently, improving system resilience and scalability.
- **Regulatory defensibility.** The immutable event log serves as evidence of when pay decisions were made, who made them, and what the workforce state was at that time -- directly relevant to EU Pay Transparency Directive compliance.

## Cons

- **Significant implementation complexity.** Event sourcing requires building event store infrastructure, projection/materialisation logic, eventual consistency handling, and idempotent event processors. This is substantially more complex than a CRUD application with a relational schema.
- **Eventual consistency in read models.** Projections are updated asynchronously, meaning the dashboard may briefly show stale data after a write. For DEI analytics (which are typically reviewed periodically, not in real-time), this is acceptable, but it adds complexity to the user experience around survey submission and goal updates.
- **Event schema evolution.** As the platform evolves, event schemas will change. Managing event versioning (upcasting old events to new schemas) requires careful design and tooling. Demographic category changes (e.g., adding neurodiversity) must be handled as new event types or versioned payloads.
- **Debugging and operational complexity.** Diagnosing data issues requires replaying event streams rather than querying a single table. Operational tooling for event stores is less mature than for relational databases.
- **Crypto-shredding complexity.** Managing per-employee encryption keys at scale (potentially hundreds of thousands of keys) requires a robust key management infrastructure. Key rotation, backup, and disaster recovery add operational burden.
- **Team skill requirements.** Event sourcing and CQRS patterns are less widely understood than traditional relational design. The development team must have or acquire expertise in these patterns.
- **Overkill for early stage.** For an MVP serving a small number of organisations, the infrastructure overhead of event sourcing may delay time-to-market compared to a simpler relational approach.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| Event store | EventStoreDB (purpose-built) or PostgreSQL with append-only table + triggers |
| Key management | HashiCorp Vault (Transit secrets engine for per-employee encryption keys) |
| Event bus | Apache Kafka or NATS JetStream for event distribution to projectors |
| Projection database | PostgreSQL for transactional projections; ClickHouse or DuckDB for analytical projections |
| Framework | Axon Framework (Java/Kotlin) or custom TypeScript with @event-driven-io patterns |
| Serialisation | JSON with schema registry (e.g., Confluent Schema Registry or custom) for event versioning |
| Monitoring | Event store lag monitoring; projection rebuild tooling; dead letter queue for failed projections |
| Statistical engine | Python services subscribing to CompensationChanged and DemographicUpdated events for pay equity analysis |

---

## Migration and Scaling Considerations

- **Event store partitioning.** Partition the event store by `organisation_id` for multi-tenant deployments. EventStoreDB supports stream-per-aggregate natively; PostgreSQL requires table partitioning.
- **Projection rebuild.** All projections must be rebuildable from the event store at any time. This is both a feature (corrections, new analytics dimensions) and an operational requirement (disaster recovery, schema changes). Automated projection rebuild tooling is essential.
- **Event store growth.** Append-only stores grow indefinitely. Implement event archival (move old events to cold storage after N years) and snapshotting (periodically capture aggregate state to avoid replaying full history).
- **Migration from CRUD.** If starting with Suggestion 1 (normalised relational) and later migrating to event sourcing, the migration path involves: (1) introduce event publishing alongside CRUD writes (dual-write), (2) build projections from events, (3) validate projection accuracy against CRUD tables, (4) switch reads to projections, (5) switch writes to event-first.
- **Multi-region deployment.** Event stores can be replicated across regions using Kafka MirrorMaker or EventStoreDB's built-in replication. Projections are rebuilt locally in each region from the replicated event stream.
- **GDPR right to erasure at scale.** Crypto-shredding is more scalable than physical deletion for event-sourced systems, but requires key management infrastructure that scales with employee count. Plan for key lifecycle management (creation, rotation, destruction, backup) from day one.
- **Scaling projections independently.** Each projection can be scaled independently -- the compliance reporting projection might run on a small instance, while the real-time dashboard projection might need a larger, faster database. CQRS enables this naturally.
