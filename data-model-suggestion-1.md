# Data Model Suggestion 1: Normalized Relational (PostgreSQL)

> Project: Developer Metrics & OKR Tracking (Candidate #297)
> Approach: Fully normalized relational schema on PostgreSQL

---

## Summary

A traditional normalized relational database design using PostgreSQL, modelled around the core domains of the application: source code events, DORA metrics, OKR goal hierarchies, teams, developer experience surveys, and notification workflows. Every entity is in third normal form (3NF) with explicit foreign key relationships, referential integrity constraints, and clear separation of concerns across schema modules.

This approach prioritises data consistency, query flexibility, and alignment with well-understood PostgreSQL tooling and ecosystem support.

---

## Core Domains and Entities

### Domain 1: Organisation and Teams

```sql
CREATE TABLE organisations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) UNIQUE NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL,
    parent_team_id  UUID REFERENCES teams(id),  -- supports team hierarchy
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, slug)
);

CREATE TABLE team_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id),
    user_id         UUID NOT NULL REFERENCES users(id),
    role            VARCHAR(50) NOT NULL DEFAULT 'member',  -- member, lead, manager
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    left_at         TIMESTAMPTZ,
    UNIQUE (team_id, user_id, joined_at)
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    email           VARCHAR(255) NOT NULL,
    display_name    VARCHAR(255),
    role            VARCHAR(50) NOT NULL DEFAULT 'viewer',  -- viewer, editor, admin
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (organisation_id, email)
);
```

### Domain 2: Data Source Integrations

```sql
CREATE TABLE integrations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    provider        VARCHAR(50) NOT NULL,  -- github, gitlab, jira, slack, pagerduty
    external_id     VARCHAR(255),          -- org/workspace ID on the provider
    access_token    TEXT,                   -- encrypted OAuth token
    refresh_token   TEXT,                   -- encrypted
    scopes          TEXT[],
    status          VARCHAR(20) NOT NULL DEFAULT 'active',
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
    language        VARCHAR(50),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (integration_id, external_id)
);
```

### Domain 3: Source Code Events (Git & CI/CD)

```sql
CREATE TABLE commits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    sha             VARCHAR(40) NOT NULL,
    author_email    VARCHAR(255),
    author_id       UUID REFERENCES users(id),
    message         TEXT,
    committed_at    TIMESTAMPTZ NOT NULL,
    lines_added     INTEGER DEFAULT 0,
    lines_removed   INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, sha)
);

CREATE TABLE pull_requests (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    external_id     VARCHAR(255) NOT NULL,
    number          INTEGER NOT NULL,
    title           TEXT,
    author_id       UUID REFERENCES users(id),
    state           VARCHAR(20) NOT NULL,  -- open, merged, closed
    base_branch     VARCHAR(255),
    head_branch     VARCHAR(255),
    lines_added     INTEGER DEFAULT 0,
    lines_removed   INTEGER DEFAULT 0,
    commit_count    INTEGER DEFAULT 0,
    first_commit_at TIMESTAMPTZ,          -- coding start
    opened_at       TIMESTAMPTZ NOT NULL,
    first_review_at TIMESTAMPTZ,          -- review start
    approved_at     TIMESTAMPTZ,
    merged_at       TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    deployed_at     TIMESTAMPTZ,          -- deploy completion
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (repository_id, external_id)
);

CREATE TABLE pull_request_reviews (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pull_request_id UUID NOT NULL REFERENCES pull_requests(id),
    reviewer_id     UUID REFERENCES users(id),
    state           VARCHAR(20) NOT NULL,  -- approved, changes_requested, commented
    submitted_at    TIMESTAMPTZ NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE deployments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    repository_id   UUID NOT NULL REFERENCES repositories(id),
    environment     VARCHAR(50) NOT NULL,  -- production, staging
    external_id     VARCHAR(255),
    sha             VARCHAR(40),
    status          VARCHAR(20) NOT NULL,  -- success, failure, rollback
    started_at      TIMESTAMPTZ NOT NULL,
    finished_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE deployment_commits (
    deployment_id   UUID NOT NULL REFERENCES deployments(id),
    commit_id       UUID NOT NULL REFERENCES commits(id),
    PRIMARY KEY (deployment_id, commit_id)
);

CREATE TABLE incidents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    integration_id  UUID REFERENCES integrations(id),  -- pagerduty, opsgenie, etc.
    external_id     VARCHAR(255),
    title           TEXT,
    severity        VARCHAR(20),   -- critical, high, medium, low
    environment     VARCHAR(50) DEFAULT 'production',
    caused_by_deployment_id UUID REFERENCES deployments(id),
    opened_at       TIMESTAMPTZ NOT NULL,
    acknowledged_at TIMESTAMPTZ,
    resolved_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Domain 4: DORA Metric Snapshots

```sql
CREATE TABLE dora_snapshots (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id                 UUID NOT NULL REFERENCES teams(id),
    period_start            DATE NOT NULL,
    period_end              DATE NOT NULL,
    granularity             VARCHAR(10) NOT NULL,  -- daily, weekly, monthly
    deployment_frequency    NUMERIC(10,4),  -- deploys per day
    lead_time_seconds       NUMERIC(12,2), -- median seconds from first commit to deploy
    mttr_seconds            NUMERIC(12,2), -- mean time to restore in seconds
    change_failure_rate     NUMERIC(5,4),  -- ratio 0.0 to 1.0
    reliability_score       NUMERIC(5,4),  -- DORA 5th metric (optional)
    calculated_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (team_id, period_start, period_end, granularity)
);

CREATE TABLE cycle_time_breakdowns (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dora_snapshot_id    UUID NOT NULL REFERENCES dora_snapshots(id),
    coding_time_seconds NUMERIC(12,2),    -- first commit to PR open
    review_time_seconds NUMERIC(12,2),    -- PR open to approved
    deploy_time_seconds NUMERIC(12,2),    -- approved to deployed
    total_time_seconds  NUMERIC(12,2),
    pr_count            INTEGER NOT NULL DEFAULT 0
);
```

### Domain 5: OKR Hierarchy

```sql
CREATE TABLE okr_periods (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    name            VARCHAR(100) NOT NULL,    -- "Q3 2026", "H2 2026"
    start_date      DATE NOT NULL,
    end_date        DATE NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'active',  -- draft, active, closed
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE objectives (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    okr_period_id   UUID NOT NULL REFERENCES okr_periods(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    parent_id       UUID REFERENCES objectives(id),  -- cascading objectives
    title           TEXT NOT NULL,
    description     TEXT,
    owner_id        UUID REFERENCES users(id),
    status          VARCHAR(20) NOT NULL DEFAULT 'on_track',
    progress        NUMERIC(5,2) DEFAULT 0.0,  -- 0 to 100
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE key_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    objective_id    UUID NOT NULL REFERENCES objectives(id),
    title           TEXT NOT NULL,
    description     TEXT,
    metric_type     VARCHAR(30) NOT NULL,
    -- metric_type: manual, deployment_frequency, lead_time, mttr,
    --              change_failure_rate, cycle_time, custom_query
    target_value    NUMERIC(12,4) NOT NULL,
    current_value   NUMERIC(12,4) DEFAULT 0,
    start_value     NUMERIC(12,4) DEFAULT 0,
    unit            VARCHAR(50),                -- "deploys/day", "hours", "%"
    direction       VARCHAR(10) DEFAULT 'up',   -- up (higher is better), down
    confidence      NUMERIC(5,2),               -- AI-predicted attainment %
    owner_id        UUID REFERENCES users(id),
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE key_result_updates (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    key_result_id   UUID NOT NULL REFERENCES key_results(id),
    value           NUMERIC(12,4) NOT NULL,
    source          VARCHAR(30) NOT NULL,   -- manual, automated, ai_prediction
    note            TEXT,
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE okr_dora_links (
    key_result_id   UUID NOT NULL REFERENCES key_results(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    dora_metric     VARCHAR(50) NOT NULL,   -- deployment_frequency, lead_time, etc.
    PRIMARY KEY (key_result_id)
);
```

### Domain 6: Developer Experience Surveys

```sql
CREATE TABLE survey_campaigns (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    title           VARCHAR(255) NOT NULL,
    status          VARCHAR(20) NOT NULL DEFAULT 'draft',
    scheduled_at    TIMESTAMPTZ,
    closed_at       TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE survey_questions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    campaign_id     UUID NOT NULL REFERENCES survey_campaigns(id),
    question_text   TEXT NOT NULL,
    question_type   VARCHAR(20) NOT NULL,  -- likert_5, likert_7, free_text, multiple_choice
    sort_order      INTEGER DEFAULT 0
);

CREATE TABLE survey_responses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    question_id     UUID NOT NULL REFERENCES survey_questions(id),
    team_id         UUID NOT NULL REFERENCES teams(id),
    respondent_id   UUID REFERENCES users(id),  -- nullable for anonymity
    value_numeric   INTEGER,
    value_text      TEXT,
    submitted_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Domain 7: AI Analysis and Reports

```sql
CREATE TABLE ai_analyses (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    team_id         UUID REFERENCES teams(id),
    analysis_type   VARCHAR(50) NOT NULL,
    -- analysis_type: root_cause, okr_draft, executive_summary,
    --               sprint_risk, bottleneck_detection
    prompt          TEXT,
    result          TEXT NOT NULL,
    model_version   VARCHAR(50),
    context_window  TEXT,       -- serialised context used
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE executive_reports (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organisation_id UUID NOT NULL REFERENCES organisations(id),
    okr_period_id   UUID REFERENCES okr_periods(id),
    title           VARCHAR(255) NOT NULL,
    content         TEXT NOT NULL,   -- markdown or HTML narrative
    generated_by    VARCHAR(20) NOT NULL,  -- ai, manual
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Domain 8: Benchmarks

```sql
CREATE TABLE industry_benchmarks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source          VARCHAR(100) NOT NULL,   -- "DORA 2025", "LinearB Dataset"
    metric          VARCHAR(50) NOT NULL,
    tier            VARCHAR(20) NOT NULL,    -- elite, high, medium, low
    value_low       NUMERIC(12,4),
    value_high      NUMERIC(12,4),
    unit            VARCHAR(50),
    year            INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Entity Relationship Summary

```
Organisation
  |-- Teams (hierarchical via parent_team_id)
  |     |-- Team Members --> Users
  |     |-- Repositories --> Integration
  |     |     |-- Commits
  |     |     |-- Pull Requests --> PR Reviews
  |     |     |-- Deployments --> Deployment Commits --> Commits
  |     |-- DORA Snapshots --> Cycle Time Breakdowns
  |     |-- Objectives (hierarchical via parent_id)
  |           |-- Key Results --> Key Result Updates
  |           |-- OKR-DORA Links
  |-- Integrations (GitHub, GitLab, Jira, Slack, PagerDuty)
  |-- Incidents --> Deployments (caused_by)
  |-- Survey Campaigns --> Questions --> Responses
  |-- OKR Periods --> Objectives
  |-- AI Analyses
  |-- Executive Reports
  |-- Industry Benchmarks
```

---

## Pros

- **Strong consistency guarantees.** ACID transactions ensure OKR progress updates, DORA calculations, and team membership changes are always consistent. No risk of partial writes or stale reads.
- **Mature tooling and ecosystem.** PostgreSQL has outstanding ORM support (SQLAlchemy, Prisma, TypeORM, Drizzle), migration tools (Flyway, Alembic, Prisma Migrate), monitoring (pg_stat_statements), and hosting options (AWS RDS, Supabase, Neon).
- **Query flexibility.** Complex analytical queries joining DORA snapshots to OKR key results to team hierarchies are straightforward with SQL JOINs. No need for specialised query languages.
- **Referential integrity.** Foreign keys prevent orphaned records (e.g., a key result referencing a deleted objective). Critical for financial and executive reporting accuracy.
- **Well-understood operational model.** PostgreSQL backup, replication, failover, and performance tuning are well-documented and widely understood by operations teams.
- **Apache DevLake precedent.** The leading open-source DORA platform (Apache DevLake) uses a normalised relational schema, validating this approach for the domain.

## Cons

- **Schema rigidity.** Adding new metric types, integration sources, or survey question formats requires DDL migrations. Fast iteration during early product development may be slowed by migration overhead.
- **Aggregation performance at scale.** Computing DORA metrics from raw events (millions of commits, PRs, deployments) requires careful indexing and potentially materialised views. Real-time dashboards may need pre-computed snapshots.
- **Time-series queries are not native.** Period-over-period trend queries (week-over-week lead time) require manual window functions or materialised views. No built-in time-bucketing, continuous aggregation, or automatic partitioning by time.
- **Hierarchical query complexity.** Recursive CTEs are needed for team hierarchies and cascading OKR rollups. These work but are less elegant and performant than graph-native alternatives.
- **Webhook event volume.** High-volume GitHub/GitLab webhook ingestion (thousands of events per minute for large organisations) may stress a single PostgreSQL instance without careful partitioning or write-ahead buffering.

---

## Technology Recommendations

| Layer | Technology |
|-------|-----------|
| Database | PostgreSQL 16+ |
| ORM / Query | Prisma or Drizzle ORM (TypeScript), SQLAlchemy (Python) |
| Migrations | Prisma Migrate, Flyway, or Alembic |
| Connection pooling | PgBouncer or Supabase connection pooler |
| Caching | Redis for dashboard query caching |
| Hosting | Supabase, Neon, AWS RDS, or self-hosted |

---

## Migration and Scaling Considerations

- **Partitioning.** Partition `commits`, `pull_requests`, and `deployments` by `committed_at` / `opened_at` / `started_at` using PostgreSQL native declarative partitioning. Monthly partitions are appropriate for most organisations.
- **Materialised views.** Pre-compute `dora_snapshots` and `cycle_time_breakdowns` via scheduled jobs rather than computing on the fly for dashboard queries. Refresh daily or on webhook batch completion.
- **Read replicas.** Route dashboard and reporting queries to read replicas to avoid contention with webhook ingestion writes.
- **Index strategy.** Composite indexes on (team_id, period_start) for DORA lookups, (repository_id, opened_at) for PR queries, and (objective_id, sort_order) for OKR trees. Use BRIN indexes on timestamp columns for large tables.
- **Multi-tenancy.** Use `organisation_id` column with row-level security (RLS) policies for tenant isolation. Alternatively, schema-per-tenant for larger deployments.
- **Future migration path.** If time-series query volume outgrows PostgreSQL, the `dora_snapshots` and event tables can be migrated to TimescaleDB (a PostgreSQL extension) without changing the application layer significantly.
