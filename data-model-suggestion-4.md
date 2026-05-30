# Data Model Suggestion 4: Time-Series Core with Graph-Enhanced OKR Layer

> Project: Developer Metrics & OKR Tracking (Candidate #297)
> Approach: TimescaleDB hypertables for metric trends + PostgreSQL with ltree/recursive structures for team and OKR hierarchies

---

## Summary

A specialty architecture that recognises the two fundamentally different data shapes in this application and optimises each independently:

1. **Time-series metrics** (DORA data points, cycle times, deployment events, survey scores) are stored in TimescaleDB hypertables with automatic partitioning, continuous aggregates, and retention policies. This is the high-volume, append-mostly, time-bucketed data that drives dashboards and trend analysis.

2. **Hierarchical goal structures** (team trees, OKR cascades, objective-to-key-result-to-DORA-metric linkages) are stored using PostgreSQL's `ltree` extension and materialised path patterns, with recursive CTEs for traversal. This is the low-volume, deeply connected data that drives OKR rollups and organisational views.

Both run on a single PostgreSQL instance with the TimescaleDB extension, avoiding the operational overhead of separate database engines while delivering specialised performance for each data shape.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                 PostgreSQL + TimescaleDB Extension              │
│                                                                 │
│  ┌──────────────────────┐    ┌───────────────────────────────┐  │
│  │   Time-Series Layer  │    │   Hierarchical / Graph Layer  │  │
│  │   (Hypertables)      │    │   (ltree + relational)        │  │
│  │                      │    │                               │  │
│  │  metric_events       │    │  organisations                │  │
│  │  deployment_events   │    │  teams (ltree path)           │  │
│  │  pr_lifecycle_events │    │  objectives (ltree path)      │  │
│  │  incident_events     │    │  key_results                  │  │
│  │  survey_scores       │    │  okr_dora_bindings            │  │
│  │                      │    │  users                        │  │
│  │  Continuous Aggs:    │    │  integrations                 │  │
│  │  dora_daily          │    │  repositories                 │  │
│  │  dora_weekly         │    │                               │  │
│  │  dora_monthly        │    │                               │  │
│  └──────────────────────┘    └───────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  Cross-Cutting: AI analyses, executive reports,          │   │
│  │  benchmarks, webhook_events (partitioned)                │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## Time-Series Layer: Hypertables

### Core Metric Events

```sql
-- Enable extensions
CREATE EXTENSION IF NOT EXISTS timescaledb;
CREATE EXTENSION IF NOT EXISTS ltree;

-- Raw metric observations (the foundational time-series table)
CREATE TABLE metric_events (
    time            TIMESTAMPTZ NOT NULL,
    organisation_id UUID NOT NULL,
    team_id         UUID NOT NULL,
    repository_id   UUID,
    metric_name     VARCHAR(80) NOT NULL,
    -- metric_name: 'deployment_frequency', 'lead_time_seconds',
    --   'mttr_seconds', 'change_failure_rate', 'coding_time_seconds',
    --   'review_time_seconds', 'deploy_time_seconds', 'pr_size_lines',
    --   'review_turnaround_seconds', 'deploy_success_rate',
    --   'reliability_score', 'dx_satisfaction', 'dx_tooling'
    value           DOUBLE PRECISION NOT NULL,
    tags            JSONB NOT NULL DEFAULT '{}',
    -- tags: { "environment": "production", "source": "github",
    --         "pr_number": 42, "severity": "high" }
    source_event_id UUID           -- link back to the originating event
);

SELECT create_hypertable('metric_events', 'time',
    chunk_time_interval => INTERVAL '1 week',
    partitioning_column => 'organisation_id',
    number_partitions => 4
);

CREATE INDEX idx_metric_team_name_time
    ON metric_events (team_id, metric_name, time DESC);
CREATE INDEX idx_metric_org_time
    ON metric_events (organisation_id, time DESC);
```

### Deployment Events

```sql
CREATE TABLE deployment_events (
    time            TIMESTAMPTZ NOT NULL,
    organisation_id UUID NOT NULL,
    team_id         UUID NOT NULL,
    repository_id   UUID NOT NULL,
    deployment_id   UUID NOT NULL,
    environment     VARCHAR(50) NOT NULL,
    sha             VARCHAR(40),
    status          VARCHAR(20) NOT NULL,   -- started, succeeded, failed, rolled_back
    duration_sec    NUMERIC(12,2),
    commit_count    INTEGER,
    pr_numbers      INTEGER[],
    details         JSONB NOT NULL DEFAULT '{}'
);

SELECT create_hypertable('deployment_events', 'time',
    chunk_time_interval => INTERVAL '1 month'
);
```

### PR Lifecycle Events

```sql
CREATE TABLE pr_lifecycle_events (
    time            TIMESTAMPTZ NOT NULL,  -- event timestamp
    organisation_id UUID NOT NULL,
    team_id         UUID NOT NULL,
    repository_id   UUID NOT NULL,
    pr_id           UUID NOT NULL,
    pr_number       INTEGER NOT NULL,
    event_type      VARCHAR(30) NOT NULL,
    -- event_type: 'opened', 'first_review', 'approved',
    --             'merged', 'closed', 'deployed'
    author_id       UUID,
    reviewer_id     UUID,
    details         JSONB NOT NULL DEFAULT '{}'
);

SELECT create_hypertable('pr_lifecycle_events', 'time',
    chunk_time_interval => INTERVAL '1 month'
);

CREATE INDEX idx_pr_lifecycle_pr ON pr_lifecycle_events (pr_id, time);
```

### Incident Events

```sql
CREATE TABLE incident_events (
    time            TIMESTAMPTZ NOT NULL,
    organisation_id UUID NOT NULL,
    team_id         UUID,
    incident_id     UUID NOT NULL,
    event_type      VARCHAR(30) NOT NULL,
    -- event_type: 'opened', 'acknowledged', 'resolved',
    --             'linked_to_deployment'
    severity        VARCHAR(20),
    environment     VARCHAR(50),
    deployment_id   UUID,
    details         JSONB NOT NULL DEFAULT '{}'
);

SELECT create_hypertable('incident_events', 'time',
    chunk_time_interval => INTERVAL '1 month'
);
```

### Survey Score Events

```sql
CREATE TABLE survey_score_events (
    time            TIMESTAMPTZ NOT NULL,
    organisation_id UUID NOT NULL,
    team_id         UUID NOT NULL,
    campaign_id     UUID NOT NULL,
    question_id     VARCHAR(50) NOT NULL,
    category        VARCHAR(50),    -- dx_satisfaction, dx_tooling, dx_process
    score           DOUBLE PRECISION NOT NULL,
    response_count  INTEGER NOT NULL DEFAULT 1
);

SELECT create_hypertable('survey_score_events', 'time',
    chunk_time_interval => INTERVAL '3 months'
);
```

---

## Continuous Aggregates (Automatic Rollups)

```sql
-- Daily DORA metrics rollup (auto-refreshed by TimescaleDB)
CREATE MATERIALIZED VIEW dora_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    organisation_id,
    team_id,
    metric_name,
    avg(value) AS avg_value,
    percentile_agg(value) AS percentile_data,  -- for median/p95 extraction
    count(*) AS sample_count,
    min(value) AS min_value,
    max(value) AS max_value
FROM metric_events
GROUP BY bucket, organisation_id, team_id, metric_name;

SELECT add_continuous_aggregate_policy('dora_daily',
    start_offset => INTERVAL '3 days',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour'
);

-- Weekly DORA metrics rollup (aggregates from daily)
CREATE MATERIALIZED VIEW dora_weekly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 week', bucket) AS bucket,
    organisation_id,
    team_id,
    metric_name,
    avg(avg_value) AS avg_value,
    sum(sample_count) AS sample_count,
    min(min_value) AS min_value,
    max(max_value) AS max_value
FROM dora_daily
GROUP BY time_bucket('1 week', bucket), organisation_id, team_id, metric_name;

SELECT add_continuous_aggregate_policy('dora_weekly',
    start_offset => INTERVAL '2 weeks',
    end_offset => INTERVAL '1 day',
    schedule_interval => INTERVAL '1 day'
);

-- Monthly DORA metrics rollup
CREATE MATERIALIZED VIEW dora_monthly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', bucket) AS bucket,
    organisation_id,
    team_id,
    metric_name,
    avg(avg_value) AS avg_value,
    sum(sample_count) AS sample_count,
    min(min_value) AS min_value,
    max(max_value) AS max_value
FROM dora_weekly
GROUP BY time_bucket('1 month', bucket), organisation_id, team_id, metric_name;

SELECT add_continuous_aggregate_policy('dora_monthly',
    start_offset => INTERVAL '2 months',
    end_offset => INTERVAL '1 week',
    schedule_interval => INTERVAL '1 week'
);
```

### Retention Policies

```sql
-- Keep raw metric_events for 6 months, then rely on continuous aggregates
SELECT add_retention_policy('metric_events', INTERVAL '6 months');
SELECT add_retention_policy('pr_lifecycle_events', INTERVAL '1 year');
SELECT add_retention_policy('deployment_events', INTERVAL '2 years');
SELECT add_retention_policy('incident_events', INTERVAL '2 years');

-- Continuous aggregates are kept indefinitely (small data volume)
```

### Compression Policies

```sql
-- Compress raw metric_events older than 1 week (10-20x compression)
ALTER TABLE metric_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'organisation_id, team_id, metric_name',
    timescaledb.compress_orderby = 'time DESC'
);

SELECT add_compression_policy('metric_events', INTERVAL '1 week');
```

---

## Hierarchical / Graph Layer: ltree + Relational

### Teams with Materialised Path

```sql
CREATE TABLE organisations (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name        VARCHAR(255) NOT NULL,
    slug        VARCHAR(100) UNIQUE NOT NULL,
    settings    JSONB NOT NULL DEFAULT '{}',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    parent_team_id  UUID REFERENCES teams(id),
    path            LTREE NOT NULL,
    -- path examples: 'eng', 'eng.platform', 'eng.platform.infra'
    depth           INTEGER GENERATED ALWAYS AS (nlevel(path)) STORED,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE INDEX idx_teams_path_gist ON teams USING gist (path);
CREATE INDEX idx_teams_path_btree ON teams USING btree (path);

-- Example queries enabled by ltree:
-- All sub-teams of 'eng':        SELECT * FROM teams WHERE path <@ 'eng';
-- Direct children of 'eng':      SELECT * FROM teams WHERE path ~ 'eng.*{1}';
-- Ancestors of 'eng.platform':   SELECT * FROM teams WHERE 'eng.platform' <@ path;
-- Depth-2 teams:                 SELECT * FROM teams WHERE nlevel(path) = 2;
```

### OKR Hierarchy with ltree

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
    path            LTREE NOT NULL,
    -- path examples: 'obj_cto_velocity',
    --                'obj_cto_velocity.obj_platform_deploy_freq',
    --                'obj_cto_velocity.obj_platform_deploy_freq.obj_infra_cicd'
    title           TEXT NOT NULL,
    description     TEXT,
    owner_id        UUID REFERENCES users(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'on_track',
    progress        NUMERIC(5,2) DEFAULT 0.0,
    ai_confidence   NUMERIC(5,2),
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_objectives_path_gist ON objectives USING gist (path);

-- Cascading OKR queries:
-- All child objectives:   SELECT * FROM objectives WHERE path <@ 'obj_cto_velocity';
-- Rollup progress:        SELECT avg(progress) FROM objectives WHERE path <@ 'obj_cto_velocity';
-- Alignment tree:         SELECT * FROM objectives WHERE path ~ 'obj_cto_velocity.*'
--                         ORDER BY path;

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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### OKR-to-DORA Bindings

```sql
CREATE TABLE okr_dora_bindings (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key_result_id   UUID NOT NULL REFERENCES key_results(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    metric_name     VARCHAR(80) NOT NULL,
    -- metric_name matches metric_events.metric_name
    auto_update     BOOLEAN NOT NULL DEFAULT true,
    target_period   TSTZRANGE NOT NULL,
    -- target_period: '[2026-04-01, 2026-07-01)'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (key_result_id)
);

-- This binding allows automatic KR progress updates:
-- SELECT avg(value) FROM dora_daily
-- WHERE team_id = binding.team_id
--   AND metric_name = binding.metric_name
--   AND bucket <@ binding.target_period;
```

---

## Cross-Cutting Entities

### Users and Integrations

```sql
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',
    identity_map    JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);

CREATE TABLE team_members (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id     UUID NOT NULL REFERENCES teams(id),
    user_id     UUID NOT NULL REFERENCES users(id),
    role        VARCHAR(50) NOT NULL DEFAULT 'member',
    joined_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    left_at     TIMESTAMPTZ,
    UNIQUE (team_id, user_id, joined_at)
);

CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    provider        VARCHAR(50) NOT NULL,
    external_id     VARCHAR(255),
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
    config          JSONB NOT NULL DEFAULT '{}',
    credentials     JSONB NOT NULL DEFAULT '{}',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE repositories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    integration_id  UUID NOT NULL REFERENCES integrations(id),
    team_id         UUID REFERENCES teams(id),
    external_id     VARCHAR(255) NOT NULL,
    name            VARCHAR(255) NOT NULL,
    url             TEXT,
    default_branch  VARCHAR(100) DEFAULT 'main',
    metadata        JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (integration_id, external_id)
);
```

### AI Analysis and Reports

```sql
CREATE TABLE ai_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    team_id         UUID REFERENCES teams(id),
    analysis_type   VARCHAR(50) NOT NULL,
    input_context   JSONB NOT NULL,
    result          JSONB NOT NULL,
    model_version   VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE executive_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    okr_period_id   UUID REFERENCES okr_periods(id),
    title           VARCHAR(255) NOT NULL,
    content_md      TEXT NOT NULL,
    content_data    JSONB NOT NULL DEFAULT '{}',
    generated_by    VARCHAR(20) NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Benchmarks (Reference Data)

```sql
CREATE TABLE industry_benchmarks (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source      VARCHAR(100) NOT NULL,
    year        INTEGER NOT NULL,
    metric_name VARCHAR(80) NOT NULL,   -- matches metric_events.metric_name
    tiers       JSONB NOT NULL,
    unit        VARCHAR(50),
    UNIQUE (source, year, metric_name)
);
```

### Webhook Event Archive

```sql
CREATE TABLE webhook_events (
    time            TIMESTAMPTZ NOT NULL,
    integration_id  UUID NOT NULL,
    provider        VARCHAR(50) NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    external_id     VARCHAR(255),
    payload         JSONB NOT NULL,
    processed       BOOLEAN NOT NULL DEFAULT false
);

SELECT create_hypertable('webhook_events', 'time',
    chunk_time_interval => INTERVAL '1 week'
);

SELECT add_retention_policy('webhook_events', INTERVAL '3 months');
SELECT add_compression_policy('webhook_events', INTERVAL '1 day');
```

---

## Example Queries

### DORA Dashboard: Weekly Lead Time Trend

```sql
SELECT
    bucket,
    avg_value AS avg_lead_time_sec,
    avg_value / 3600.0 AS avg_lead_time_hours
FROM dora_weekly
WHERE team_id = :team_id
    AND metric_name = 'lead_time_seconds'
    AND bucket >= now() - INTERVAL '12 weeks'
ORDER BY bucket;
```

### Period-over-Period Comparison

```sql
WITH current_period AS (
    SELECT metric_name, avg(avg_value) AS current_avg
    FROM dora_weekly
    WHERE team_id = :team_id
        AND bucket >= now() - INTERVAL '4 weeks'
    GROUP BY metric_name
),
previous_period AS (
    SELECT metric_name, avg(avg_value) AS previous_avg
    FROM dora_weekly
    WHERE team_id = :team_id
        AND bucket >= now() - INTERVAL '8 weeks'
        AND bucket < now() - INTERVAL '4 weeks'
    GROUP BY metric_name
)
SELECT
    c.metric_name,
    c.current_avg,
    p.previous_avg,
    ((c.current_avg - p.previous_avg) / NULLIF(p.previous_avg, 0)) * 100 AS pct_change
FROM current_period c
JOIN previous_period p USING (metric_name);
```

### OKR Cascade Rollup with DORA Data

```sql
-- Roll up progress for all objectives under a top-level objective
WITH objective_tree AS (
    SELECT o.id, o.title, o.path, o.progress, o.ai_confidence
    FROM objectives o
    WHERE o.path <@ :root_objective_path
        AND o.okr_period_id = :period_id
    ORDER BY o.path
)
SELECT
    ot.id,
    ot.title,
    nlevel(ot.path) - nlevel(:root_objective_path) AS depth,
    ot.progress,
    ot.ai_confidence,
    -- Latest DORA-linked KR value
    (SELECT me.avg_value FROM dora_weekly me
     JOIN okr_dora_bindings b ON b.team_id = me.team_id
        AND b.metric_name = me.metric_name
     JOIN key_results kr ON kr.id = b.key_result_id
     WHERE kr.objective_id = ot.id
     ORDER BY me.bucket DESC LIMIT 1
    ) AS latest_dora_value
FROM objective_tree ot;
```

### Team Hierarchy with DORA Tier

```sql
SELECT
    t.id,
    t.name,
    t.path,
    nlevel(t.path) AS depth,
    ds.dora_tier,
    ds.deployment_frequency,
    ds.lead_time_median_sec
FROM teams t
LEFT JOIN LATERAL (
    SELECT
        CASE
            WHEN avg(CASE WHEN metric_name = 'deployment_frequency' THEN avg_value END) >= 7 THEN 'elite'
            WHEN avg(CASE WHEN metric_name = 'deployment_frequency' THEN avg_value END) >= 1 THEN 'high'
            WHEN avg(CASE WHEN metric_name = 'deployment_frequency' THEN avg_value END) >= 0.14 THEN 'medium'
            ELSE 'low'
        END AS dora_tier,
        avg(CASE WHEN metric_name = 'deployment_frequency' THEN avg_value END) AS deployment_frequency,
        avg(CASE WHEN metric_name = 'lead_time_seconds' THEN avg_value END) AS lead_time_median_sec
    FROM dora_weekly
    WHERE team_id = t.id
        AND bucket >= now() - INTERVAL '4 weeks'
) ds ON true
WHERE t.path <@ :root_team_path
ORDER BY t.path;
```

---

## Pros

- **Purpose-built for time-series metrics.** TimescaleDB hypertables automatically partition by time, compress old data (10-20x), and enforce retention policies. Period-over-period DORA comparisons, trend analysis, and percentile calculations are first-class operations, not workarounds.
- **Continuous aggregates eliminate batch jobs.** Daily, weekly, and monthly DORA rollups are maintained automatically by TimescaleDB. No scheduled ETL jobs, no stale materialised views, no cron-based refresh scripts. Dashboards query pre-computed aggregates and render in milliseconds.
- **Elegant hierarchical queries.** The `ltree` extension makes OKR cascade rollups, team subtree queries, and ancestor lookups performant and readable. No recursive CTEs needed for common hierarchy operations.
- **Natural data lifecycle management.** Raw metric events can be retained for 6 months, PR lifecycle events for 1 year, and aggregates indefinitely. Automatic retention policies prevent unbounded storage growth -- critical for a metrics platform that ingests data continuously.
- **Single database engine.** Despite being specialised for two different data shapes, everything runs on PostgreSQL + extensions. One backup strategy, one connection pool, one monitoring stack.
- **High ingestion throughput.** TimescaleDB handles millions of metric events per second. GitHub/GitLab webhook volumes (even for large organisations) are well within capacity.
- **OKR-to-DORA binding is elegant.** The `okr_dora_bindings` table creates a clean, queryable link between OKR key results and the time-series metrics that drive them. Automatic KR progress updates are a simple JOIN query.

## Cons

- **TimescaleDB extension dependency.** Requires the TimescaleDB extension, which may not be available on all PostgreSQL hosting providers. Supabase does not support TimescaleDB; Neon support is limited. Self-hosting or Timescale Cloud may be required.
- **ltree learning curve.** The `ltree` extension has a specialised query syntax (`<@`, `~`, `?`, `@>`) that is unfamiliar to most developers. Documentation is sparse compared to standard SQL patterns.
- **ltree path maintenance.** When a team or objective is moved in the hierarchy, all descendant paths must be updated. This requires careful transaction handling and is more complex than simply updating a `parent_id` foreign key.
- **Two mental models.** Developers must think in time-series patterns (bucketing, aggregation, retention) for metrics and in graph/tree patterns (paths, ancestors, descendants) for OKRs. This dual mental model increases onboarding time.
- **Continuous aggregate limitations.** TimescaleDB continuous aggregates have restrictions: no JOINs in the aggregate definition, limited window function support, and the refresh policy introduces a small lag. Complex DORA calculations (e.g., percentile across multiple metrics) may need to be done in application code.
- **Vendor-specific features.** TimescaleDB compression, retention policies, and continuous aggregates are not standard PostgreSQL. If migrating away from TimescaleDB, these features would need to be reimplemented.
- **Overkill for small teams.** For organisations with fewer than 50 developers, the volume of metric events is low enough that standard PostgreSQL with scheduled materialised view refreshes would perform equally well. The TimescaleDB overhead (hypertable chunking, compression workers) adds complexity without proportional benefit at small scale.

---

## Technology Recommendations

| Layer | Technology | Notes |
|-------|-----------|-------|
| Database | PostgreSQL 16 + TimescaleDB 2.x | Single instance with both extensions |
| Extensions | `timescaledb`, `ltree`, `pgcrypto` | All available in standard PostgreSQL distributions |
| Hosting | Timescale Cloud, Aiven, or self-hosted | Supabase/Neon do not fully support TimescaleDB |
| ORM | Kysely or raw SQL | ORMs have limited TimescaleDB and ltree support |
| Migrations | dbmate or sqitch | SQL-first tools that handle extension DDL |
| Dashboard | Grafana or custom (Next.js + Chart.js) | Grafana has native TimescaleDB data source |
| Caching | Redis | For hot dashboard query results |

---

## Migration and Scaling Considerations

- **Starting simple.** Begin with Suggestion 1 or 3 (standard PostgreSQL) for the MVP. When metric event volume reaches the point where dashboard query latency degrades (typically 10M+ rows in event tables), add the TimescaleDB extension and convert the metric tables to hypertables. This is a non-destructive operation.
- **Chunk interval tuning.** Start with 1-week chunks for `metric_events` and 1-month chunks for other hypertables. Adjust based on actual ingestion rates and query patterns. Smaller chunks = faster compression and retention, but more chunk management overhead.
- **Space partitioning.** For multi-tenant deployments, add `organisation_id` as a space partition dimension on hypertables. This ensures tenant data is co-located on disk, improving query performance for tenant-scoped dashboards.
- **ltree path convention.** Establish a path naming convention early (e.g., `org_slug.team_slug` for teams, `period_slug.objective_slug` for OKRs). Document it. Inconsistent paths create debugging nightmares.
- **Backup strategy.** TimescaleDB hypertables are backed up with standard `pg_dump` but restore times can be long for large datasets. Consider Timescale Cloud's managed backups or WAL-based continuous archiving for production.
- **Grafana integration.** If using Grafana for dashboards, the TimescaleDB data source provides native support for time-bucketing, continuous aggregate queries, and variable-based team/metric selection. This can accelerate MVP dashboard development significantly.
- **Read replicas.** Route dashboard queries to a streaming replica. TimescaleDB continuous aggregates work on replicas, so dashboards can query pre-computed rollups without loading the primary.
