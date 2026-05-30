# Data Model Suggestion 2: Event-Sourced / CQRS Approach

> Project: Developer Metrics & OKR Tracking (Candidate #297)
> Approach: Event sourcing with Command Query Responsibility Segregation (CQRS)

---

## Summary

An event-sourced architecture where every state change in the system is captured as an immutable event in an append-only event store. The write side (command model) persists events representing webhook deliveries, DORA metric observations, OKR state transitions, and survey responses. The read side (query model) materialises purpose-built projections optimised for dashboards, reports, and AI analysis.

This approach treats engineering activity data as a permanent, auditable event stream -- a natural fit for a platform whose core purpose is to answer "what happened, when, and why?" about software delivery performance.

---

## Architecture Overview

```
                    ┌──────────────┐
  GitHub/GitLab ──> │  Command API │ ──> Event Store (append-only)
  Jira/Slack    ──> │  (writes)    │         │
  PagerDuty     ──> │              │         │
  Manual input  ──> └──────────────┘         │
                                             ▼
                                      Event Projectors
                                    /    |     |     \
                                   ▼     ▼     ▼      ▼
                              DORA    OKR    Team    Survey
                              View   View   View    View
                                \     |      |      /
                                 ▼    ▼      ▼     ▼
                              ┌──────────────────────┐
                              │     Query API        │
                              │  (read-optimised)    │
                              └──────────────────────┘
```

---

## Event Store Schema

### Core Event Table

```sql
-- The append-only event store (PostgreSQL or EventStoreDB)
CREATE TABLE events (
    event_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       VARCHAR(255) NOT NULL,    -- e.g. "repo:abc123", "team:xyz", "okr:obj-456"
    stream_type     VARCHAR(50) NOT NULL,     -- repository, team, objective, key_result, survey
    event_type      VARCHAR(100) NOT NULL,    -- see event catalogue below
    event_version   INTEGER NOT NULL,         -- optimistic concurrency per stream
    payload         JSONB NOT NULL,           -- event-specific data
    metadata        JSONB NOT NULL DEFAULT '{}',  -- correlation_id, causation_id, actor_id
    organisation_id UUID NOT NULL,            -- tenant partition key
    occurred_at     TIMESTAMPTZ NOT NULL,     -- when the event happened in the real world
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, event_version)
);

CREATE INDEX idx_events_stream ON events (stream_id, event_version);
CREATE INDEX idx_events_type ON events (event_type, occurred_at);
CREATE INDEX idx_events_org_time ON events (organisation_id, occurred_at);
```

### Event Catalogue

```
── Source Code Events ──────────────────────────────────────
CommitPushed           { sha, repo_id, author, message, lines_added, lines_removed, timestamp }
PullRequestOpened      { pr_id, repo_id, author, title, base_branch, head_branch, opened_at }
PullRequestReviewed    { pr_id, reviewer, state, submitted_at }
PullRequestMerged      { pr_id, merged_by, merged_at, commit_sha }
PullRequestClosed      { pr_id, closed_at, reason }

── Deployment Events ──────────────────────────────────────
DeploymentStarted      { deployment_id, repo_id, environment, sha, started_at }
DeploymentSucceeded    { deployment_id, finished_at }
DeploymentFailed       { deployment_id, finished_at, error }
DeploymentRolledBack   { deployment_id, rollback_to_sha, rolled_back_at }

── Incident Events ────────────────────────────────────────
IncidentOpened         { incident_id, title, severity, environment, opened_at }
IncidentAcknowledged   { incident_id, acknowledged_by, acknowledged_at }
IncidentResolved       { incident_id, resolved_at, resolution_note }
IncidentLinkedToDeploy { incident_id, deployment_id }

── OKR Events ─────────────────────────────────────────────
OKRPeriodCreated       { period_id, name, start_date, end_date }
OKRPeriodActivated     { period_id, activated_at }
OKRPeriodClosed        { period_id, closed_at, final_score }
ObjectiveCreated       { objective_id, period_id, team_id, title, owner_id }
ObjectiveUpdated       { objective_id, changes: { title?, description?, status? } }
ObjectiveScoreChanged  { objective_id, old_score, new_score, source }
KeyResultCreated       { kr_id, objective_id, title, metric_type, target, start_value, unit }
KeyResultValueRecorded { kr_id, value, source, note, recorded_at }
KeyResultLinkedToDORA  { kr_id, team_id, dora_metric }

── Survey Events ──────────────────────────────────────────
SurveyCampaignCreated  { campaign_id, title, questions }
SurveyCampaignOpened   { campaign_id, opened_at }
SurveyResponseSubmitted { campaign_id, question_id, team_id, value, submitted_at }
SurveyCampaignClosed   { campaign_id, closed_at }

── Team Events ────────────────────────────────────────────
TeamCreated            { team_id, name, parent_team_id }
TeamMemberAdded        { team_id, user_id, role, joined_at }
TeamMemberRemoved      { team_id, user_id, left_at }

── Integration Events ─────────────────────────────────────
IntegrationConnected   { integration_id, provider, scopes }
IntegrationSynced      { integration_id, records_processed, synced_at }
IntegrationDisconnected { integration_id, reason }

── AI Events ──────────────────────────────────────────────
AIAnalysisRequested    { analysis_id, type, team_id, prompt }
AIAnalysisCompleted    { analysis_id, result, model_version }
ExecutiveReportGenerated { report_id, period_id, content }
```

---

## Read Model Projections

### DORA Metrics Projection

```sql
-- Materialised by the DORA Projector from deployment, PR, and incident events
CREATE TABLE dora_metrics_view (
    team_id                 UUID NOT NULL,
    period_start            DATE NOT NULL,
    period_end              DATE NOT NULL,
    granularity             VARCHAR(10) NOT NULL,
    deployment_frequency    NUMERIC(10,4),
    lead_time_median_sec    NUMERIC(12,2),
    lead_time_p95_sec       NUMERIC(12,2),
    mttr_median_sec         NUMERIC(12,2),
    change_failure_rate     NUMERIC(5,4),
    total_deployments       INTEGER,
    failed_deployments      INTEGER,
    total_incidents         INTEGER,
    pr_count                INTEGER,
    coding_time_median_sec  NUMERIC(12,2),
    review_time_median_sec  NUMERIC(12,2),
    deploy_time_median_sec  NUMERIC(12,2),
    last_projected_at       TIMESTAMPTZ NOT NULL,
    PRIMARY KEY (team_id, period_start, granularity)
);
```

### OKR Progress Projection

```sql
-- Materialised by the OKR Projector from OKR events
CREATE TABLE okr_progress_view (
    objective_id    UUID NOT NULL,
    period_id       UUID NOT NULL,
    team_id         UUID NOT NULL,
    title           TEXT NOT NULL,
    status          VARCHAR(20),
    progress        NUMERIC(5,2),
    key_results     JSONB NOT NULL,   -- denormalised array of KR summaries
    parent_id       UUID,
    owner_name      VARCHAR(255),
    ai_confidence   NUMERIC(5,2),     -- predicted attainment %
    last_updated_at TIMESTAMPTZ,
    PRIMARY KEY (objective_id)
);
```

### Team Dashboard Projection

```sql
-- Materialised by the Team Projector
CREATE TABLE team_dashboard_view (
    team_id             UUID PRIMARY KEY,
    team_name           VARCHAR(255),
    member_count        INTEGER,
    active_repos        INTEGER,
    open_pr_count       INTEGER,
    avg_pr_age_hours    NUMERIC(8,2),
    dora_tier           VARCHAR(20),    -- elite, high, medium, low
    current_objectives  INTEGER,
    okr_avg_progress    NUMERIC(5,2),
    last_deployment_at  TIMESTAMPTZ,
    last_incident_at    TIMESTAMPTZ,
    last_projected_at   TIMESTAMPTZ
);
```

### PR Activity Projection

```sql
-- Materialised from PR lifecycle events
CREATE TABLE pr_activity_view (
    pr_id               UUID PRIMARY KEY,
    repository_id       UUID NOT NULL,
    team_id             UUID NOT NULL,
    number              INTEGER,
    title               TEXT,
    author_name         VARCHAR(255),
    state               VARCHAR(20),
    coding_time_sec     NUMERIC(12,2),
    review_time_sec     NUMERIC(12,2),
    deploy_time_sec     NUMERIC(12,2),
    total_cycle_time_sec NUMERIC(12,2),
    reviewer_count      INTEGER,
    opened_at           TIMESTAMPTZ,
    merged_at           TIMESTAMPTZ,
    deployed_at         TIMESTAMPTZ
);
```

### Survey Results Projection

```sql
CREATE TABLE survey_results_view (
    campaign_id     UUID NOT NULL,
    team_id         UUID NOT NULL,
    question_text   TEXT NOT NULL,
    avg_score       NUMERIC(3,2),
    response_count  INTEGER,
    score_distribution JSONB,  -- { "1": 2, "2": 5, "3": 8, "4": 12, "5": 3 }
    PRIMARY KEY (campaign_id, team_id, question_text)
);
```

---

## Event Processing Pipeline

```
Webhook Ingestion
       │
       ▼
┌─────────────────┐
│  Command Handler│ ── validates, enriches, emits event
│  (per provider) │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  Event Store    │ ── append-only write (PostgreSQL / EventStoreDB)
│  (events table) │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
Sync       Async
Projectors  Projectors
    │         │
    ▼         ▼
Read DB   Notification    AI Analysis
(views)   Service         Service
          (Slack)         (root-cause, OKR drafts)
```

**Projector implementation options:**
1. **PostgreSQL LISTEN/NOTIFY** -- for low-latency, simple setups
2. **Kafka / NATS / Redis Streams** -- for high-throughput, multi-consumer setups
3. **Outbox pattern** -- events written to PostgreSQL, polled by consumers (reliable, no message broker dependency)

---

## Pros

- **Complete audit trail.** Every webhook delivery, metric change, OKR update, and AI analysis is preserved as an immutable event. "Why did our lead time spike?" can be answered by replaying events, not just querying snapshots.
- **Natural fit for webhook-driven data.** GitHub/GitLab webhooks are already events. The model maps 1:1 to the incoming data shape rather than forcing webhook payloads into normalised tables.
- **Temporal queries are first-class.** "What was our DORA tier on January 15th?" is trivially answerable by replaying events up to that date. No need for slowly-changing dimension patterns.
- **Read model flexibility.** New dashboard views, report formats, or AI analysis inputs can be added by writing new projectors without changing the write model. No schema migrations needed for new read patterns.
- **Decoupled scaling.** Write path (webhook ingestion) and read path (dashboard queries) scale independently. High-volume ingestion does not affect dashboard latency.
- **Reproducible analytics.** DORA calculations can be re-run from raw events if the calculation algorithm changes (e.g., switching from mean to median lead time). No data loss.

## Cons

- **Operational complexity.** Event sourcing requires maintaining event store, projectors, read databases, and potentially a message broker. More moving parts than a single PostgreSQL database.
- **Eventual consistency.** Read models are updated asynchronously. Dashboard data may lag behind the latest webhook by seconds to minutes depending on projector throughput. Users may see stale data after a deployment.
- **Projection rebuild cost.** If a projector has a bug or a new read model is needed, rebuilding from the full event history can take hours for organisations with years of data. Requires snapshotting strategy.
- **Event schema evolution.** Events are immutable, but their schema evolves. Version management (upcasting old events to new formats) adds complexity. Requires disciplined event versioning from day one.
- **Higher initial development cost.** More code to write: command handlers, event definitions, projectors, snapshot logic, and idempotency handling. MVP timeline is longer than with a simple CRUD approach.
- **Team skill requirements.** Event sourcing and CQRS are less widely understood than traditional relational designs. Hiring and onboarding may be more challenging.
- **Query complexity for ad-hoc analysis.** SQL-literate users cannot simply JOIN tables for ad-hoc queries. They must query the read models, which may not have the exact shape needed. Adding new views requires writing new projectors.

---

## Technology Recommendations

| Layer | Technology | Notes |
|-------|-----------|-------|
| Event Store | PostgreSQL (events table) or EventStoreDB | PostgreSQL is simpler to operate; EventStoreDB is purpose-built |
| Event Bus | NATS JetStream or Apache Kafka | For async projector fan-out; NATS for simplicity, Kafka for scale |
| Read Database | PostgreSQL (projections) | Same engine, different schema; simpler ops |
| Dashboard Cache | Redis | For hot dashboard data |
| Command API | Node.js/TypeScript or Go | Event handler logic |
| Projectors | Node.js/TypeScript workers | One per read model |
| Event Serialisation | JSON (JSONB in PostgreSQL) | With JSON Schema validation |

---

## Migration and Scaling Considerations

- **Event store partitioning.** Partition the `events` table by `organisation_id` and `occurred_at` (range partitioning on month). Keeps individual tenant queries fast and enables per-tenant archival.
- **Snapshots.** Store periodic aggregate snapshots (e.g., team DORA state every 1000 events) to avoid replaying the full stream on projector restart. Implement as a separate `snapshots` table.
- **Idempotency.** Webhook events from GitHub/GitLab may be delivered multiple times. Use `(stream_id, external_event_id)` deduplication at the command handler to prevent duplicate events.
- **Event versioning.** Use a `schema_version` field in the event payload. Projectors must handle all versions via upcasters that transform old event shapes to the current format.
- **Cold storage.** Events older than a configurable retention period (e.g., 2 years) can be archived to object storage (S3) and removed from the hot event store. Projections remain unaffected since they are already materialised.
- **Migration from relational.** If starting with Suggestion 1 (normalised relational) and migrating later, the relational tables can be treated as the initial event projection while a backfill process generates synthetic events from the existing data.
- **Multi-region.** The append-only event store is inherently replication-friendly. Events can be replicated across regions with eventual consistency, and regional read models can project locally for low-latency dashboards.
