# Data Model Suggestion 4: Bitemporal Relational + Graph Hybrid

> Project: Diversity, Equity & Inclusion Platform (Candidate #416)
> Approach: Bitemporal relational model for audit-grade workforce history combined with a graph layer for relationship-driven DEI analytics

---

## Summary

This architecture combines two specialised data modelling techniques that address core DEI platform requirements better than any single approach. A **bitemporal relational model** (PostgreSQL with SQL:2011 temporal support) captures workforce data with two independent time dimensions -- *valid time* (when a fact was true in reality) and *transaction time* (when the system recorded it) -- providing an auditable, legally defensible record of every workforce state change. A **property graph layer** (Apache AGE extension for PostgreSQL, or Neo4j) models the relationship-rich aspects of the DEI domain: organisational hierarchies, mentorship networks, ERG membership graphs, reporting chains, and team composition patterns that drive inclusion outcomes. The combination is particularly powerful for this domain because DEI analysis is fundamentally about understanding both *how workforce data changes over time* (bitemporal) and *how people are connected and positioned within organisational networks* (graph).

---

## Key Entities and Relationships

### Part 1: Bitemporal Relational Model

Bitemporal tables maintain two time ranges on every row:

- `valid_period`: When the fact is/was true in reality (e.g., "this employee was in Engineering from 2024-03-01 to 2025-06-15")
- `system_period`: When the database recorded this information (for audit trail and correction handling)

```sql
-- Enable temporal extensions
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Bitemporal employee record
CREATE TABLE employee_bt (
    id                UUID NOT NULL,
    organisation_id   UUID NOT NULL REFERENCES organisation(id),
    employee_ref      TEXT NOT NULL,
    department_id     UUID,
    job_title         TEXT,
    job_level         INT,
    location_country  CHAR(2),
    location_city     TEXT,
    status            TEXT NOT NULL,

    -- Valid time: when this state was true in reality
    valid_from        DATE NOT NULL,
    valid_to          DATE NOT NULL DEFAULT '9999-12-31',

    -- System time: when this was recorded in the database
    system_from       TIMESTAMPTZ NOT NULL DEFAULT now(),
    system_to         TIMESTAMPTZ NOT NULL DEFAULT 'infinity',

    -- Constraints
    PRIMARY KEY (id, valid_from, system_from),
    EXCLUDE USING GIST (
        id WITH =,
        daterange(valid_from, valid_to) WITH &&,
        tstzrange(system_from, system_to) WITH &&
    )
);

-- History table for system-time versioning (auto-populated by triggers)
CREATE TABLE employee_bt_history (LIKE employee_bt INCLUDING ALL);

-- Trigger to maintain system-time versioning
CREATE OR REPLACE FUNCTION maintain_system_time()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        -- Close the old system period
        INSERT INTO employee_bt_history
        SELECT OLD.* ;
        NEW.system_from := now();
        NEW.system_to := 'infinity';
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO employee_bt_history
        SELECT OLD.*;
        RETURN OLD;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER employee_bt_versioning
    BEFORE UPDATE OR DELETE ON employee_bt
    FOR EACH ROW EXECUTE FUNCTION maintain_system_time();
```

**Bitemporal Compensation**

```sql
CREATE TABLE compensation_bt (
    id                UUID NOT NULL,
    employee_id       UUID NOT NULL,
    organisation_id   UUID NOT NULL REFERENCES organisation(id),
    base_salary       NUMERIC(12,2) NOT NULL,
    currency          CHAR(3) NOT NULL,
    bonus_target      NUMERIC(12,2),
    equity_value      NUMERIC(12,2),
    total_comp        NUMERIC(12,2),
    decision_type     TEXT,
    decision_maker_id UUID,          -- who approved this pay decision (for accountability)
    justification     TEXT,          -- documented reasoning (for pay transparency)

    valid_from        DATE NOT NULL,
    valid_to          DATE NOT NULL DEFAULT '9999-12-31',
    system_from       TIMESTAMPTZ NOT NULL DEFAULT now(),
    system_to         TIMESTAMPTZ NOT NULL DEFAULT 'infinity',

    PRIMARY KEY (id, valid_from, system_from),
    EXCLUDE USING GIST (
        employee_id WITH =,
        daterange(valid_from, valid_to) WITH &&,
        tstzrange(system_from, system_to) WITH &&
    )
);

CREATE TABLE compensation_bt_history (LIKE compensation_bt INCLUDING ALL);

CREATE TRIGGER compensation_bt_versioning
    BEFORE UPDATE OR DELETE ON compensation_bt
    FOR EACH ROW EXECUTE FUNCTION maintain_system_time();
```

**Bitemporal Demographics (with Encryption)**

```sql
CREATE TABLE demographic_bt (
    employee_id       UUID NOT NULL,
    organisation_id   UUID NOT NULL REFERENCES organisation(id),

    -- Encrypted demographic fields
    demographics_encrypted BYTEA,       -- AES-256 encrypted JSON payload
    encryption_key_id TEXT NOT NULL,     -- reference to Vault key

    -- Unencrypted metadata for querying aggregates
    -- (these are the broad categories, not fine-grained values)
    has_gender        BOOLEAN DEFAULT false,
    has_ethnicity     BOOLEAN DEFAULT false,
    has_disability    BOOLEAN DEFAULT false,
    has_veteran       BOOLEAN DEFAULT false,
    consent_status    TEXT NOT NULL CHECK (consent_status IN ('granted', 'revoked', 'pending')),
    legal_basis       TEXT NOT NULL,
    data_source       TEXT NOT NULL,

    valid_from        DATE NOT NULL,
    valid_to          DATE NOT NULL DEFAULT '9999-12-31',
    system_from       TIMESTAMPTZ NOT NULL DEFAULT now(),
    system_to         TIMESTAMPTZ NOT NULL DEFAULT 'infinity',

    PRIMARY KEY (employee_id, valid_from, system_from)
);

CREATE TABLE demographic_bt_history (LIKE demographic_bt INCLUDING ALL);
```

**Bitemporal Queries: Time-Travel Examples**

```sql
-- What was the workforce composition of Engineering on 2025-01-01?
-- (as we knew it on 2025-03-01, even if data was corrected later)
SELECT e.*, d.demographics_encrypted
FROM employee_bt e
JOIN demographic_bt d ON d.employee_id = e.id
WHERE e.department_id = :engineering_dept_id
  AND e.valid_from <= '2025-01-01' AND e.valid_to > '2025-01-01'      -- valid at that date
  AND e.system_from <= '2025-03-01' AND e.system_to > '2025-03-01';   -- as known on that date

-- What was the gender pay gap when we ran the Q4 2024 pay equity analysis?
-- Reproduce the exact data that was available at analysis time
SELECT c.*, e.job_level, e.department_id
FROM compensation_bt c
JOIN employee_bt e ON e.id = c.employee_id
  AND e.valid_from <= '2024-12-31' AND e.valid_to > '2024-12-31'
  AND e.system_from <= '2025-01-15' AND e.system_to > '2025-01-15'
WHERE c.valid_from <= '2024-12-31' AND c.valid_to > '2024-12-31'
  AND c.system_from <= '2025-01-15' AND c.system_to > '2025-01-15';

-- Show all corrections to an employee's compensation record
-- (system_from changes indicate when corrections were recorded)
SELECT *
FROM (
    SELECT * FROM compensation_bt WHERE employee_id = :emp_id
    UNION ALL
    SELECT * FROM compensation_bt_history WHERE employee_id = :emp_id
) all_versions
ORDER BY system_from;
```

### Part 2: Graph Layer for Relationship Analytics

The graph layer models entities and relationships that are naturally graph-shaped: reporting chains, mentorship networks, ERG communities, team compositions, and inter-departmental connections.

**Option A: Apache AGE (PostgreSQL Extension)**

```sql
-- Apache AGE runs inside PostgreSQL -- no separate database
CREATE EXTENSION IF NOT EXISTS age;
SELECT create_graph('dei_graph');

-- Create nodes from relational data
SELECT * FROM cypher('dei_graph', $$
    CREATE (e:Employee {
        id: 'emp-001',
        department: 'Engineering',
        job_level: 5,
        location: 'London',
        gender: 'Female',
        ethnicity: 'Asian',
        hire_date: '2022-03-15'
    })
$$) AS (v agtype);

-- Create relationship edges
SELECT * FROM cypher('dei_graph', $$
    MATCH (mentor:Employee {id: 'emp-042'}), (mentee:Employee {id: 'emp-001'})
    CREATE (mentor)-[:MENTORS {
        programme: 'Women in Leadership',
        started: '2025-01-10',
        status: 'active'
    }]->(mentee)
$$) AS (e agtype);

SELECT * FROM cypher('dei_graph', $$
    MATCH (member:Employee {id: 'emp-001'}), (erg:ERG {name: 'Women in Tech'})
    CREATE (member)-[:MEMBER_OF {
        joined: '2022-04-01',
        role: 'co-lead'
    }]->(erg)
$$) AS (e agtype);

SELECT * FROM cypher('dei_graph', $$
    MATCH (report:Employee {id: 'emp-001'}), (manager:Employee {id: 'emp-023'})
    CREATE (report)-[:REPORTS_TO {
        since: '2024-06-01'
    }]->(manager)
$$) AS (e agtype);
```

**Option B: Neo4j (Dedicated Graph Database)**

```cypher
// Organisational hierarchy with diversity annotations
CREATE (dept:Department {id: 'eng', name: 'Engineering', org_id: 'org-001'})
CREATE (team:Team {id: 'eng-ml', name: 'ML Platform', dept_id: 'eng'})
CREATE (team)-[:PART_OF]->(dept)

// Employee nodes with demographic properties
CREATE (e:Employee {
    id: 'emp-001',
    job_level: 5,
    location: 'London',
    gender: 'Female',
    ethnicity: 'Asian',
    tenure_years: 4
})
CREATE (e)-[:WORKS_IN]->(team)
CREATE (e)-[:LOCATED_IN]->(:Location {country: 'GB', city: 'London'})

// Reporting chain
CREATE (e)-[:REPORTS_TO]->(mgr:Employee {id: 'emp-023'})

// Mentorship network
CREATE (e)<-[:MENTORS {programme: 'Women in Leadership', since: '2025-01-10'}]-
       (mentor:Employee {id: 'emp-042'})

// ERG membership
CREATE (e)-[:MEMBER_OF {role: 'co-lead', since: '2022-04-01'}]->
       (erg:ERG {name: 'Women in Tech', type: 'affinity'})

// Pay equity alert chain (who approved problematic pay decisions)
CREATE (decision:PayDecision {
    id: 'pd-789',
    type: 'merit',
    date: '2025-06-01',
    amount: 5000,
    equity_flag: 'review'
})
CREATE (e)-[:RECEIVED]->(decision)
CREATE (decision)-[:APPROVED_BY]->(approver:Employee {id: 'emp-023'})
```

**Graph Queries for DEI-Specific Insights**

```cypher
// 1. Manager inclusion score: What is the demographic diversity
//    of each manager's direct reports?
MATCH (mgr:Employee)<-[:REPORTS_TO]-(report:Employee)
WITH mgr, collect(report) AS reports,
     count(DISTINCT report.gender) AS gender_diversity,
     count(DISTINCT report.ethnicity) AS ethnicity_diversity,
     count(report) AS team_size
RETURN mgr.id, team_size, gender_diversity, ethnicity_diversity
ORDER BY gender_diversity ASC

// 2. Mentorship network reach: Are underrepresented employees
//    connected to senior sponsors?
MATCH path = (mentee:Employee)-[:MENTORS*1..3]-(senior:Employee)
WHERE mentee.gender = 'Female'
  AND mentee.ethnicity IN ['Black', 'Hispanic']
  AND senior.job_level >= 8
RETURN mentee.id, length(path) AS degrees_to_senior, senior.id
ORDER BY degrees_to_senior

// 3. ERG community overlap: Which ERGs share the most members?
MATCH (e:Employee)-[:MEMBER_OF]->(erg1:ERG),
      (e)-[:MEMBER_OF]->(erg2:ERG)
WHERE erg1.name < erg2.name
WITH erg1.name AS group1, erg2.name AS group2, count(e) AS shared_members
RETURN group1, group2, shared_members
ORDER BY shared_members DESC

// 4. Promotion pathway analysis: Do underrepresented employees
//    have similar career path patterns to majority employees?
MATCH path = (emp:Employee)-[:PROMOTED_TO*]->(current_role)
WHERE emp.hire_date > '2020-01-01'
WITH emp, length(path) AS promotions,
     duration.between(date(emp.hire_date), date()).years AS tenure
RETURN emp.gender, emp.ethnicity,
       avg(promotions) AS avg_promotions,
       avg(toFloat(promotions)/tenure) AS promotion_velocity

// 5. Organisational network centrality: Are underrepresented
//    employees as connected as majority employees?
MATCH (e:Employee)
CALL {
    WITH e
    MATCH (e)-[r]-()
    RETURN count(r) AS connections
}
RETURN e.gender, e.ethnicity,
       avg(connections) AS avg_connections,
       percentileCont(connections, 0.5) AS median_connections
ORDER BY avg_connections
```

### Integration Between Bitemporal and Graph Layers

```
                    +-----------------------+
                    |   HRIS Integration    |
                    |   (Merge / Finch)     |
                    +-----------+-----------+
                                |
                    +-----------v-----------+
                    |   Command Handlers    |
                    |   (Write Path)        |
                    +-----------+-----------+
                                |
              +-----------------+-----------------+
              |                                   |
  +-----------v-----------+         +-------------v-----------+
  |  Bitemporal Tables    |         |    Graph Sync Service   |
  |  (PostgreSQL)         |         |    (CDC / Triggers)     |
  |                       |         +-------------+-----------+
  |  - employee_bt        |                       |
  |  - compensation_bt    |         +-------------v-----------+
  |  - demographic_bt     |         |    Graph Database       |
  |  - survey             |         |    (AGE or Neo4j)       |
  |  - pay_equity_run     |         |                         |
  +-----------+-----------+         |  - Org hierarchy        |
              |                     |  - Reporting chains     |
              |                     |  - Mentorship network   |
  +-----------v-----------+         |  - ERG membership       |
  |  Analytical Views     |         |  - Career pathways      |
  |  (Materialised)       |         +-------------------------+
  |                       |
  |  - Dashboard data     |
  |  - Compliance reports |
  |  - Pay equity results |
  +-----------------------+
```

The graph database is synchronised from the bitemporal relational tables via Change Data Capture (Debezium) or PostgreSQL triggers. The bitemporal tables are the system of record; the graph is a derived, queryable projection optimised for relationship traversal.

---

## Pros

- **Legally defensible audit trail.** Bitemporal modelling provides an exact record of what the system knew at any point in time, and when each fact was true in reality. This is critical for pay equity litigation ("what was the pay gap when the merit cycle was approved?"), GDPR data subject access requests ("what data did we hold about this person on date X?"), and regulatory audits.
- **Correction handling without data loss.** When HRIS data is corrected retroactively (e.g., an employee's hire date was entered incorrectly), the bitemporal model preserves both the original incorrect entry and the correction, along with timestamps of when each was recorded. Traditional models overwrite the old data.
- **Reproducible analyses.** Pay equity analyses can be exactly reproduced months or years later by querying the bitemporal tables at the original analysis timestamp. This eliminates the common problem of "the numbers changed since we last ran the report" caused by data corrections applied between runs.
- **Graph-native relationship queries.** DEI analysis is fundamentally about how people are positioned within organisational networks. Questions like "do underrepresented employees have equal access to senior mentors?" and "is a manager's team less diverse than comparable teams?" are naturally expressed as graph traversals, not SQL joins.
- **Mentorship and ERG network analysis.** The graph layer enables sophisticated network analysis: identifying isolated employees who lack mentorship connections, detecting ERG communities that are siloed from the broader organisation, and measuring the "reach" of sponsorship programmes.
- **Manager accountability modelling.** The graph layer can model who approved which pay decisions, creating a clear chain of accountability for compensation equity. This directly supports EU Pay Transparency Directive requirements.
- **Intersectional pathway analysis.** Graph path queries can compare career progression patterns (promotion velocity, lateral moves, access to high-visibility projects) across demographic groups, revealing structural barriers that aggregate statistics miss.

## Cons

- **Highest implementation complexity.** This approach combines two advanced data modelling techniques (bitemporal and graph), each of which independently adds significant complexity. The combination requires expertise in both temporal SQL and graph query languages.
- **Two query paradigms.** Developers must work with SQL (including temporal predicates) for the relational layer and Cypher (or Apache AGE's Cypher-in-SQL) for the graph layer. This dual-paradigm approach increases the learning curve and makes code review more demanding.
- **Graph synchronisation overhead.** Keeping the graph database in sync with the bitemporal relational tables requires a reliable CDC pipeline. Eventual consistency between the two systems means the graph may briefly lag behind the relational tables.
- **Bitemporal query complexity.** Bitemporal SQL queries are inherently more complex than standard queries. Every query must consider both valid_time and system_time ranges, which increases the cognitive load on developers and the risk of subtle bugs.
- **Storage cost.** Bitemporal tables with full history can grow significantly larger than current-state-only tables. For an organisation with 10,000 employees and 5 years of history, the bitemporal tables (including history) may be 10-50x larger than a simple current-state table.
- **Operational complexity.** Running two data systems (PostgreSQL for bitemporal + Neo4j for graph, or managing the Apache AGE extension) increases operational overhead: monitoring, backups, upgrades, and failure recovery all become more complex.
- **Graph database maturity for HR.** While graph databases are well-proven for fraud detection and recommendation engines, their application to HR and DEI analytics is less mature, with fewer ready-made tools and patterns.
- **Apache AGE limitations.** If using AGE within PostgreSQL (avoiding a separate Neo4j deployment), AGE is less mature than Neo4j and has limitations in graph algorithm support, tooling, and performance at large scale.

---

## Technology Recommendations

| Component | Recommendation |
|---|---|
| Primary database | PostgreSQL 16+ with SQL:2011 temporal features and btree_gist extension |
| Graph layer (Option A) | Apache AGE extension for PostgreSQL (single-engine deployment) |
| Graph layer (Option B) | Neo4j 5+ (dedicated graph DB for advanced graph algorithms) |
| CDC pipeline | Debezium for PostgreSQL-to-Neo4j sync (if using Option B) |
| Key management | HashiCorp Vault for per-employee demographic encryption keys |
| Bitemporal ORM support | Custom repository layer; few ORMs support bitemporal natively (consider jOOQ for Java/Kotlin or raw SQL) |
| Graph visualisation | Neo4j Bloom or custom D3.js/Sigma.js for organisational network visualisation |
| Analytical layer | Materialised views on bitemporal tables for dashboard queries; DuckDB for ad-hoc analytical queries |
| Statistical engine | Python (statsmodels, NetworkX for graph metrics) consuming data from both layers |
| HRIS integration | Merge or Finch unified HRIS API; write to bitemporal tables with explicit valid_from dates |

---

## Migration and Scaling Considerations

- **Start with bitemporal, add graph later.** The bitemporal relational model can be implemented first as the system of record. The graph layer can be added as a derived projection when the platform requires relationship-based analytics (mentorship matching, ERG analysis, manager accountability). This staged approach reduces initial implementation risk.
- **Apache AGE vs. Neo4j decision.** For early-stage deployment, Apache AGE keeps everything in PostgreSQL (single operational footprint). If the platform reaches scale where graph algorithm performance matters (thousands of mentorship pairs, complex network centrality calculations), migrate to Neo4j. The Cypher query language is the same in both.
- **Bitemporal table partitioning.** Partition bitemporal tables by `organisation_id` and by time range (e.g., yearly partitions on `valid_from`). This keeps query performance manageable as history accumulates and enables per-tenant data management.
- **History table archival.** Move very old system-time history (e.g., corrections older than 7 years) to cold storage (S3/GCS with Parquet format). Maintain the valid-time dimension indefinitely for regulatory compliance but archive the system-time audit trail beyond retention requirements.
- **GDPR right to erasure.** For crypto-shredding, destroy the per-employee encryption key in Vault. The bitemporal history (both current and history tables) retains its structural integrity, but encrypted demographic payloads become permanently unreadable. Non-demographic fields (department, job level, dates) can be anonymised or retained based on legal basis.
- **Graph layer scaling.** Neo4j scales vertically for read-heavy graph traversals. For very large organisations (100,000+ employees with complex relationship networks), consider Neo4j Fabric for federated graph queries or AuraDB (Neo4j's managed service) for operational simplicity.
- **Migration from simpler models.** If starting with Suggestion 1 (normalised relational) and later upgrading to bitemporal: (1) add valid_from/valid_to columns with default values covering the existing data period, (2) add system_from/system_to columns with the migration timestamp, (3) create history tables, (4) install versioning triggers. This is a non-destructive migration that preserves all existing data.
- **Multi-region deployment.** Deploy separate bitemporal PostgreSQL instances per data residency region. The graph layer can be centralised (with pseudonymised node IDs, no PII in graph properties) or regional, depending on data residency requirements. Graph traversal queries across regions require federated graph queries if regional deployment is chosen.

---

## When to Choose This Approach

This model is the strongest choice when:

- **Regulatory defensibility is paramount.** Organisations subject to EU Pay Transparency Directive audits, CSRD assurance requirements, or pay equity litigation need the point-in-time reproduction capability that bitemporal modelling provides.
- **Relationship-driven DEI insights are a differentiator.** If the platform's competitive advantage includes network-based inclusion analysis (mentorship reach, manager team diversity, promotion pathway comparison), the graph layer provides capabilities that are extremely difficult to replicate in pure SQL.
- **The team has (or can acquire) bitemporal and graph expertise.** This is the most technically demanding approach and requires a development team comfortable with temporal SQL, graph query languages, and the operational complexity of multi-paradigm data systems.
- **The platform targets enterprise customers.** Large enterprises (5,000+ employees) with complex organisational structures, multiple mentorship programmes, and regulatory compliance requirements benefit most from the combination of bitemporal audit trails and graph-based relationship analytics.
