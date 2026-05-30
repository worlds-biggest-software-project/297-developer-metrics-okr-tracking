# Data Model Suggestion 3: Hybrid Relational + JSONB/Document Approach

> Project: Developer Metrics & OKR Tracking (Candidate #297)
> Approach: PostgreSQL with strategic JSONB columns for semi-structured and evolving data

---

## Summary

A pragmatic hybrid approach using PostgreSQL with normalised relational tables for core entities (teams, OKRs, DORA snapshots) combined with JSONB columns for semi-structured, evolving, and provider-specific data (webhook payloads, integration metadata, survey schemas, AI analysis context). This design captures the consistency and query power of a relational model while avoiding the migration burden that comes from forcing every webhook field and provider-specific attribute into rigid column definitions.

This approach is inspired by how modern engineering analytics platforms handle the diversity of data sources (GitHub, GitLab, Jira, PagerDuty each have different payload shapes) without requiring a schema migration every time a provider adds a new field.

---

## Design Principles

1. **Relational for queried fields.** Any column that appears in a WHERE clause, JOIN condition, GROUP BY, or ORDER BY gets a dedicated, typed, indexed column.
2. **JSONB for the rest.** Provider-specific fields, optional metadata, configuration blobs, and evolving schemas live in JSONB columns with GIN indexes.
3. **Validated JSONB.** CHECK constraints with `jsonb_typeof()` and application-level JSON Schema validation prevent garbage data in JSONB columns.
4. **One database engine.** Everything runs on PostgreSQL. No separate document store, no MongoDB sidecar. Operational simplicity is a feature.

---

## Schema Design

### Organisation and Teams (Fully Relational)

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings: { "default_timezone": "UTC", "fiscal_year_start": 1,
    --             "dora_calculation_method": "median", "anonymise_surveys": true }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    parent_team_id  UUID REFERENCES teams(id),
    settings        JSONB NOT NULL DEFAULT '{}',
    -- settings: { "slack_channel": "#eng-platform", "dora_target_tier": "high" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    identity_map    JSONB NOT NULL DEFAULT '{}',
    -- identity_map: { "github": "octocat", "gitlab": "user123",
    --                 "jira": "JIRAUSER456", "slack": "U0123ABC" }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

CREATE TABLE team_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    left_at         TIMESTAMPTZ,
    UNIQUE (team_id, user_id, joined_at)
);
```

### Integrations (Relational + JSONB Config)

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    provider        VARCHAR(50) NOT NULL,
    external_id     VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    credentials     JSONB NOT NULL DEFAULT '{}',   -- encrypted at rest
    -- credentials: { "access_token": "enc:...", "refresh_token": "enc:...",
    --               "webhook_secret": "enc:...", "scopes": ["repo", "read:org"] }
    config          JSONB NOT NULL DEFAULT '{}',
    -- config: { "sync_branches": ["main", "master"], "deploy_environments": ["production"],
    --           "incident_labels": ["bug", "incident"], "excluded_repos": ["*.fork"] }
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Source Code Events (Relational Core + JSONB Details)

```sql
CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES integrations(id),
    team_id         UUID REFERENCES teams(id),
    external_id     VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    url             TEXT,
    default_branch  VARCHAR(100) DEFAULT 'main',
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata: { "language": "TypeScript", "topics": ["backend", "api"],
    --             "visibility": "private", "archived": false }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (integration_id, external_id)
);

CREATE TABLE pull_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    external_id     VARCHAR(255) NOT NULL,
    number          INTEGER NOT NULL,
    title           TEXT,
    author_id       UUID REFERENCES users(id),
    state           VARCHAR(20) NOT NULL,

    -- Timestamps for cycle time calculation (relational -- always queried)
    first_commit_at TIMESTAMPTZ,
    opened_at       TIMESTAMPTZ NOT NULL,
    first_review_at TIMESTAMPTZ,
    approved_at     TIMESTAMPTZ,
    merged_at       TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    deployed_at     TIMESTAMPTZ,

    -- Computed cycle time stages (seconds, for fast aggregation)
    coding_time_sec NUMERIC(12,2) GENERATED ALWAYS AS (
        EXTRACT(EPOCH FROM (opened_at - first_commit_at))
    ) STORED,
    review_time_sec NUMERIC(12,2) GENERATED ALWAYS AS (
        EXTRACT(EPOCH FROM (COALESCE(approved_at, merged_at) - opened_at))
    ) STORED,

    -- Provider-specific details (JSONB -- varies by GitHub vs GitLab)
    details         JSONB NOT NULL DEFAULT '{}',
    -- details: { "base_branch": "main", "head_branch": "feat/new-thing",
    --            "lines_added": 142, "lines_removed": 37, "commit_count": 5,
    --            "labels": ["enhancement"], "reviewers": ["alice", "bob"],
    --            "checks_passed": true, "draft": false,
    --            "linked_issues": ["PROJ-123", "PROJ-456"] }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, external_id)
);

CREATE INDEX idx_pr_team_time ON pull_requests (repository_id, opened_at);
CREATE INDEX idx_pr_details ON pull_requests USING gin (details);

CREATE TABLE deployments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    external_id     VARCHAR(255),
    environment     VARCHAR(50) NOT NULL,
    sha             VARCHAR(40),
    status          VARCHAR(20) NOT NULL,
    started_at      TIMESTAMPTZ NOT NULL,
    finished_at     TIMESTAMPTZ,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details: { "pipeline_id": "...", "trigger": "push", "duration_sec": 185,
    --            "stages": [{"name": "build", "duration": 60}, {"name": "deploy", "duration": 125}],
    --            "commit_count": 3, "pr_numbers": [42, 43] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    integration_id  UUID REFERENCES integrations(id),
    external_id     VARCHAR(255),
    title           TEXT,
    severity        VARCHAR(20),
    environment     VARCHAR(50) DEFAULT 'production',
    opened_at       TIMESTAMPTZ NOT NULL,
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    details         JSONB NOT NULL DEFAULT '{}',
    -- details: { "source": "pagerduty", "service": "api-gateway",
    --            "escalation_policy": "platform-team",
    --            "caused_by_deployment_id": "...",
    --            "root_cause": "memory leak in connection pool",
    --            "timeline": [...], "related_prs": [42] }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Webhook Event Log (Append-Only, JSONB-Heavy)

```sql
CREATE TABLE webhook_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES integrations(id),
    provider        VARCHAR(50) NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    -- event_type: "push", "pull_request.opened", "deployment_status",
    --             "issue.updated", "incident.triggered"
    external_id     VARCHAR(255),
    payload         JSONB NOT NULL,           -- raw webhook payload (full fidelity)
    processed       BOOLEAN NOT NULL DEFAULT false,
    processed_at    TIMESTAMPTZ,
    error           TEXT,
    received_at     TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (received_at);

-- Monthly partitions
CREATE TABLE webhook_events_2026_05 PARTITION OF webhook_events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
CREATE TABLE webhook_events_2026_06 PARTITION OF webhook_events
    FOR VALUES FROM ('2026-06-01') TO ('2026-07-01');

CREATE INDEX idx_webhook_unprocessed ON webhook_events (processed, received_at)
    WHERE NOT processed;
```

### DORA Metrics (Fully Relational -- Always Queried)

```sql
CREATE TABLE dora_snapshots (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id                 UUID NOT NULL REFERENCES teams(id),
    period_start            DATE NOT NULL,
    period_end              DATE NOT NULL,
    granularity             VARCHAR(10) NOT NULL,

    -- Core DORA four keys
    deployment_frequency    NUMERIC(10,4),
    lead_time_median_sec    NUMERIC(12,2),
    lead_time_p95_sec       NUMERIC(12,2),
    mttr_median_sec         NUMERIC(12,2),
    change_failure_rate     NUMERIC(5,4),

    -- Cycle time breakdown
    coding_time_median_sec  NUMERIC(12,2),
    review_time_median_sec  NUMERIC(12,2),
    deploy_time_median_sec  NUMERIC(12,2),

    -- Counts for context
    total_deployments       INTEGER DEFAULT 0,
    total_prs_merged        INTEGER DEFAULT 0,
    total_incidents         INTEGER DEFAULT 0,

    -- Extended metrics (JSONB -- evolving set)
    extended_metrics        JSONB NOT NULL DEFAULT '{}',
    -- extended_metrics: { "reliability_score": 0.95,
    --                     "pr_size_median_lines": 87,
    --                     "review_turnaround_p50_sec": 3600,
    --                     "deploy_success_rate": 0.97,
    --                     "space_satisfaction": 4.2 }

    dora_tier               VARCHAR(20),  -- elite, high, medium, low
    calculated_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, period_start, period_end, granularity)
);

CREATE INDEX idx_dora_team_period ON dora_snapshots (team_id, period_start DESC);
```

### OKR Hierarchy (Fully Relational)

```sql
CREATE TABLE okr_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(100) NOT NULL,
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE objectives (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    okr_period_id   UUID NOT NULL REFERENCES okr_periods(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    parent_id       UUID REFERENCES objectives(id),
    title           TEXT NOT NULL,
    description     TEXT,
    owner_id        UUID REFERENCES users(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'on_track',
    progress        NUMERIC(5,2) DEFAULT 0.0,
    sort_order      INTEGER DEFAULT 0,
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- metadata: { "ai_suggested": true, "tags": ["reliability", "velocity"],
    --             "linked_epics": ["PROJ-100"], "notes": "..." }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE key_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    objective_id    UUID NOT NULL REFERENCES objectives(id),
    title           TEXT NOT NULL,
    metric_type     VARCHAR(30) NOT NULL,
    target_value    NUMERIC(12,4) NOT NULL,
    current_value   NUMERIC(12,4) DEFAULT 0,
    start_value     NUMERIC(12,4) DEFAULT 0,
    unit            VARCHAR(50),
    direction       VARCHAR(10) DEFAULT 'up',
    confidence      NUMERIC(5,2),
    owner_id        UUID REFERENCES users(id),
    sort_order      INTEGER DEFAULT 0,
    dora_link       JSONB,
    -- dora_link: { "team_id": "...", "metric": "deployment_frequency",
    --              "auto_update": true }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE key_result_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key_result_id   UUID NOT NULL REFERENCES key_results(id),
    value           NUMERIC(12,4) NOT NULL,
    source          VARCHAR(30) NOT NULL,
    note            TEXT,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Developer Experience Surveys (Relational + JSONB Schema)

```sql
CREATE TABLE survey_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    question_schema JSONB NOT NULL,
    -- question_schema: [
    --   { "id": "q1", "text": "How productive do you feel?",
    --     "type": "likert_5", "required": true },
    --   { "id": "q2", "text": "What slows you down the most?",
    --     "type": "free_text", "required": false },
    --   { "id": "q3", "text": "Rate your build tooling",
    --     "type": "likert_7", "category": "dx_tooling" }
    -- ]
    scheduled_at    TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE survey_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES survey_campaigns(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    respondent_hash VARCHAR(64),   -- anonymised hash for dedup without identification
    answers         JSONB NOT NULL,
    -- answers: { "q1": 4, "q2": "Flaky CI pipeline", "q3": 6 }
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### AI Analysis (JSONB-Heavy by Nature)

```sql
CREATE TABLE ai_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    team_id         UUID REFERENCES teams(id),
    analysis_type   VARCHAR(50) NOT NULL,
    trigger         VARCHAR(30) NOT NULL DEFAULT 'manual',  -- manual, scheduled, anomaly
    input_context   JSONB NOT NULL,
    -- input_context: { "period": "2026-05-01/2026-05-15",
    --                  "metrics_snapshot": {...}, "recent_incidents": [...],
    --                  "pr_outliers": [...] }
    result          JSONB NOT NULL,
    -- result: { "summary": "Lead time increased 40% due to...",
    --           "root_causes": [...], "recommendations": [...],
    --           "confidence": 0.85 }
    model_version   VARCHAR(50),
    tokens_used     INTEGER,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE executive_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    okr_period_id   UUID REFERENCES okr_periods(id),
    title           VARCHAR(255) NOT NULL,
    content_md      TEXT NOT NULL,
    content_data    JSONB NOT NULL DEFAULT '{}',
    -- content_data: { "charts": [...], "highlights": [...],
    --                 "team_scores": {...} }
    generated_by    VARCHAR(20) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Benchmarks

```sql
CREATE TABLE industry_benchmarks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source          VARCHAR(100) NOT NULL,
    year            INTEGER NOT NULL,
    metric          VARCHAR(50) NOT NULL,
    tiers           JSONB NOT NULL,
    -- tiers: { "elite": { "min": 7, "description": "Multiple deploys per day" },
    --          "high":  { "min": 1, "max": 7, "description": "Daily to weekly" },
    --          "medium": { "min": 0.14, "max": 1 },
    --          "low":   { "max": 0.14, "description": "Less than once per month" } }
    unit            VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source, year, metric)
);
```

---

## When JSONB vs. When Relational: Decision Guide

| Use Case | Storage | Rationale |
|----------|---------|-----------|
| Team hierarchy, OKR tree structure | Relational FK | Always queried, JOINed, aggregated |
| DORA four-key values | Relational columns | Always filtered, sorted, compared |
| PR lifecycle timestamps | Relational columns | Cycle time computed from these |
| PR labels, linked issues, reviewer names | JSONB `details` | Provider-specific, not always queried |
| Raw webhook payloads | JSONB `payload` | Full fidelity preservation, rarely queried directly |
| Survey questions definition | JSONB `question_schema` | Schema varies per campaign, defined by user |
| Survey answers | JSONB `answers` | Shape matches question schema, aggregated in application |
| User identity across providers | JSONB `identity_map` | Set of provider mappings varies per user |
| AI analysis input/output | JSONB | Unstructured by nature, schema evolves with AI model changes |
| Integration credentials/config | JSONB | Provider-specific configuration, rarely JOINed |
| Extended/experimental metrics | JSONB `extended_metrics` | New metrics added without DDL migration |
| Organisation/team settings | JSONB `settings` | User-configurable, arbitrary key-value |

---

## Indexing Strategy

```sql
-- GIN indexes for JSONB query support
CREATE INDEX idx_pr_details_gin ON pull_requests USING gin (details);
CREATE INDEX idx_user_identity_gin ON users USING gin (identity_map);
CREATE INDEX idx_deployment_details_gin ON deployments USING gin (details);

-- Partial GIN index: only index specific JSONB keys that are queried
CREATE INDEX idx_pr_labels ON pull_requests USING gin ((details -> 'labels'));
CREATE INDEX idx_pr_linked_issues ON pull_requests USING gin ((details -> 'linked_issues'));

-- Expression index: extract JSONB value for efficient lookup
CREATE INDEX idx_user_github ON users ((identity_map ->> 'github'));

-- Standard B-tree indexes on relational columns
CREATE INDEX idx_dora_team_period ON dora_snapshots (team_id, period_start DESC);
CREATE INDEX idx_pr_repo_opened ON pull_requests (repository_id, opened_at);
CREATE INDEX idx_objectives_period ON objectives (okr_period_id, team_id);
CREATE INDEX idx_kr_objective ON key_results (objective_id);
```

---

## Pros

- **Pragmatic balance.** Core query paths (DORA dashboards, OKR trees, cycle time breakdowns) use typed, indexed relational columns for full SQL power. Provider-specific details use JSONB to avoid migration churn.
- **Single database engine.** Everything runs on PostgreSQL. No MongoDB, no Redis (unless caching is needed), no separate document store. Dramatically simpler operations, backup, and monitoring.
- **Graceful schema evolution.** New fields from GitHub API changes, new metric types, new survey question formats, or new AI analysis output shapes are absorbed by JSONB columns without DDL migrations. Only promoted to relational columns when they become query-critical.
- **Full webhook fidelity.** The `webhook_events` table stores raw payloads, enabling replay, debugging, and future extraction of fields not yet modelled. No data loss at ingestion time.
- **Generated columns.** PostgreSQL generated columns (e.g., `coding_time_sec`, `review_time_sec`) pre-compute cycle time stages from timestamps, giving dashboard-ready values without application logic.
- **JSONB validation.** CHECK constraints and application-level JSON Schema validation provide schema-on-write guarantees for JSONB columns, preventing garbage data while maintaining flexibility.
- **Proven at scale.** Companies like Supabase, PostHog, and GitLab use this hybrid PostgreSQL pattern in production with millions of rows. The pattern is well-documented and operationally understood.

## Cons

- **Discipline required.** The boundary between "relational column" and "JSONB field" must be actively managed. Without clear guidelines, developers may put queryable data in JSONB (slow) or stable data in relational columns (unnecessary migration work).
- **JSONB query performance.** GIN-indexed JSONB queries are significantly slower than B-tree indexed relational column queries for large datasets. Queries filtering on deeply nested JSONB paths may be slow without careful index design.
- **No cross-JSONB foreign keys.** A deployment_id stored inside `incidents.details` JSONB has no referential integrity guarantee. The application must enforce consistency for references stored in JSONB.
- **ORM friction.** Most ORMs (Prisma, TypeORM) have limited JSONB support. Complex JSONB queries often require raw SQL, reducing the benefit of using an ORM.
- **Reporting complexity.** Business intelligence tools (Metabase, Looker) can query JSONB but the experience is worse than querying flat relational columns. Executive report builders may need views that flatten JSONB into columns.
- **Storage overhead.** JSONB stores keys with every row. For high-volume tables like `webhook_events`, this adds meaningful storage overhead compared to normalised relational columns.

---

## Technology Recommendations

| Layer | Technology | Notes |
|-------|-----------|-------|
| Database | PostgreSQL 16+ | JSONB, generated columns, declarative partitioning |
| ORM | Drizzle ORM or Kysely | Better JSONB/raw SQL support than Prisma |
| JSONB Validation | Ajv (Node.js) or jsonschema (Python) | Application-level JSON Schema validation |
| Migrations | Drizzle Kit or dbmate | Lightweight, SQL-first migration tools |
| Connection Pooling | PgBouncer | Required for serverless or high-connection environments |
| Caching | Redis or PostgreSQL UNLOGGED tables | For hot dashboard data |
| Hosting | Supabase, Neon, or AWS RDS | All support JSONB indexes and partitioning |

---

## Migration and Scaling Considerations

- **Partition webhook_events by month.** This is the highest-volume table. Monthly range partitions on `received_at` enable efficient pruning of old data and fast queries on recent events.
- **Promote JSONB to columns when needed.** When a JSONB field becomes frequently queried (e.g., `details->>'lines_added'` appears in many reports), promote it to a relational column via an `ALTER TABLE ADD COLUMN` + backfill migration. The JSONB field remains for backward compatibility.
- **Materialised views for dashboards.** Create materialised views joining DORA snapshots with team hierarchy and OKR progress. Refresh on a schedule (every 15 minutes) or via trigger.
- **JSONB column compression.** PostgreSQL TOAST automatically compresses large JSONB values. For `webhook_events.payload`, this typically achieves 3-5x compression.
- **Multi-tenancy.** Use `organisation_id` with row-level security (RLS). JSONB columns do not affect RLS policies.
- **Future migration to event sourcing.** The `webhook_events` table already serves as a lightweight event log. If the system evolves toward full event sourcing (Suggestion 2), this table becomes the seed for the event store, and the relational tables become read projections.
- **TimescaleDB upgrade path.** If time-series query volume grows, the `dora_snapshots` table can be converted to a TimescaleDB hypertable without changing the rest of the schema.
