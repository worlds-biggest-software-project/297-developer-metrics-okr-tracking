# Developer Metrics & OKR Tracking — Phased Development Plan

> Project: 297-developer-metrics-okr-tracking · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesises `research.md`, `features.md`, `standards.md`, `README.md`, and `data-model-suggestion-3.md` (the chosen hybrid relational + JSONB model). It targets the MVP scope from `features.md` and stages the should-have AI features behind a working metrics engine.

---

## Product Summary

An AI-native, open-source engineering intelligence platform that collects DORA four-key metrics from GitHub and GitLab, breaks down cycle time by stage, and maps delivery data to OKRs. It serves VPs of Engineering, engineering managers, and finance partners who need defensible delivery answers without enterprise SaaS pricing. The differentiators are: cross-provider DORA aggregation (GitHub + GitLab unified), native OKR authoring linked to DORA-derived key results, an MCP server exposing metrics to AI agents, and AI-generated root-cause analysis / executive narratives. Privacy-by-design: team-level aggregation by default, individual drill-down opt-in only (GDPR posture).

**Deployment model:** Self-hosted first (Docker Compose), SaaS-ready (multi-tenant via `organisation_id` + Postgres RLS). API + web dashboard + MCP server.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22 LTS) | The integration surface (GitHub/GitLab/Jira/Slack webhooks, REST API, web dashboard) is I/O-bound and JS-ecosystem-native. Octokit, `jira.js`, Slack Bolt, and the Anthropic SDK are all first-class in TS. One language across API, workers, MCP server, and frontend reduces context-switching. |
| Runtime / framework | Fastify | Highest-throughput Node HTTP framework; first-class JSON Schema validation (aligns with OpenAPI 3.1) and built-in serialization. Webhook ingestion needs low overhead and raw-body access for HMAC signature verification, which Fastify supports cleanly via `rawBody`. |
| API contract | OpenAPI 3.1, generated from Fastify route schemas via `@fastify/swagger` | `standards.md` calls for OpenAPI 3.1 to enable SDK generation, docs portals, and MCP tool definitions. Schema-first routes give validation + spec for free. |
| Database | PostgreSQL 16 | Chosen data model (`data-model-suggestion-3.md`) is hybrid relational + JSONB on a single Postgres engine. ACID for OKR/metric consistency; JSONB + GIN for diverse provider payloads without migration churn; RLS for multi-tenancy. |
| ORM / query | Drizzle ORM | TypeScript-native, SQL-first (no heavy abstraction over analytical JOINs/window functions needed for DORA aggregation), first-class JSONB column typing, and lightweight migrations via `drizzle-kit`. |
| Migrations | drizzle-kit | Generates SQL migrations from the Drizzle schema; checked into `migrations/`. |
| Task queue | BullMQ (Redis-backed) | Webhook handlers must return fast (GitHub/GitLab require <10s, ideally <1s). Heavy work (payload normalisation, metric recomputation, LLM calls, Slack sends) runs in BullMQ workers with retry/backoff. Redis doubles as dashboard cache. |
| Cache | Redis 7 | Dashboard query caching + BullMQ broker. Single dependency serves both. |
| Scheduler | BullMQ repeatable jobs | Nightly DORA snapshot recomputation, weekly digests, survey campaign dispatch. No separate cron service needed. |
| LLM provider | Anthropic Claude (via `@anthropic-ai/sdk`), provider-abstracted | AI-native features (root-cause, OKR drafting, executive summaries) are core. Abstract behind an `LLMClient` interface so OpenAI/local models can be swapped. Prompt caching enabled for the large metric-context blocks. |
| MCP server | `@modelcontextprotocol/sdk` (TypeScript) | `standards.md` identifies an MCP server as the key unmet differentiator. Exposes `get_dora_metrics`, `get_cycle_time_breakdown`, `list_open_prs_by_age`, `get_okr_progress` over stdio + HTTP. |
| Frontend | Next.js 16 (App Router) + React + shadcn/ui + Recharts | Server components for fast dashboards; shadcn/ui for accessible primitives; Recharts for DORA trend/heatmap visualisations. Deployed alongside the API or to Vercel. |
| Auth (app) | Auth.js (NextAuth) with OIDC + credentials; SAML/SCIM deferred | `standards.md` flags OIDC/SAML/SCIM as enterprise gates. MVP ships OIDC + email/password; SAML 2.0 + SCIM 2.0 are a later phase. |
| Auth (API) | API keys (`x-api-key`) + OAuth2 bearer (JWT, RFC 7519) | Matches incumbent conventions (LinearB `x-api-key`); JWT for session-derived API access. |
| Secrets / token encryption | `libsodium` (sealed boxes) via `sodium-native` | OAuth tokens for integrations stored encrypted at rest (`access_token`/`refresh_token` columns). |
| Testing | Vitest + Supertest-style Fastify `inject` + Testcontainers (Postgres/Redis) | Vitest for unit; Fastify `.inject()` for route integration; Testcontainers for real-DB integration tests; Playwright for dashboard E2E. |
| Code quality | ESLint (typescript-eslint) + Prettier + `tsc --noEmit` | Standard TS toolchain; type-check gate in CI. |
| Package manager | pnpm (workspaces / monorepo) | Monorepo for `api`, `worker`, `web`, `mcp`, and shared `core` package. pnpm workspaces give fast, disk-efficient installs. |
| Containerisation | Docker + Docker Compose | Self-hosted target. Compose brings up Postgres, Redis, API, worker, web, MCP. |
| Key libraries | `octokit`, `@gitbeaker/node` (GitLab), `jira.js`, `@slack/bolt`, `zod`, `date-fns`, `node:crypto` (HMAC) | Domain integrations + schema validation + date math + signature verification. |

### Repository / Project Structure

A pnpm monorepo. Group by concern; each phase adds modules without restructuring.

```
developer-metrics-okr/
├── package.json                      # workspace root
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── .eslintrc.cjs / .prettierrc
├── docker-compose.yml                # postgres, redis, api, worker, web, mcp
├── Dockerfile.api
├── Dockerfile.worker
├── Dockerfile.web
├── Dockerfile.mcp
├── .env.example
├── openapi/                          # generated OpenAPI 3.1 artefacts
│   └── dora-events.schema.json       # canonical DORA event JSON Schema (Phase 3)
├── migrations/                       # drizzle-kit SQL migrations
├── packages/
│   ├── core/                         # shared domain logic (no I/O)
│   │   ├── src/
│   │   │   ├── schema/               # Drizzle table definitions + zod models
│   │   │   ├── dora/                 # metric calculators (pure functions)
│   │   │   ├── cycle-time/
│   │   │   ├── okr/                   # OKR progress + attainment logic
│   │   │   ├── events/               # canonical event normalisation
│   │   │   ├── llm/                   # LLMClient interface + Anthropic impl
│   │   │   └── types.ts
│   │   └── tests/
│   ├── api/                          # Fastify HTTP service
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── plugins/              # auth, rawBody, db, redis, swagger
│   │   │   ├── routes/
│   │   │   │   ├── webhooks/         # github, gitlab, jira
│   │   │   │   ├── metrics/          # dora, cycle-time
│   │   │   │   ├── okrs/
│   │   │   │   ├── teams/
│   │   │   │   ├── surveys/
│   │   │   │   ├── ai/               # root-cause, summaries, okr-draft
│   │   │   │   └── benchmarks/
│   │   │   └── lib/
│   │   └── tests/
│   ├── worker/                       # BullMQ processors + schedulers
│   │   ├── src/
│   │   │   ├── queues.ts
│   │   │   ├── processors/          # normalise, recompute, notify, ai
│   │   │   └── schedules/           # nightly snapshots, weekly digest
│   │   └── tests/
│   ├── mcp/                          # MCP server
│   │   ├── src/server.ts
│   │   └── tests/
│   └── web/                          # Next.js dashboard
│       ├── app/
│       ├── components/
│       └── tests/                    # Playwright E2E
└── scripts/
    ├── seed.ts                       # demo org + synthetic events
    └── backfill.ts                   # historical pull for new integrations
```

---

## Phase 1: Foundation — Monorepo, Schema, Config, Auth Skeleton

### Purpose
Establish the buildable, testable skeleton: the pnpm monorepo, the PostgreSQL schema (hybrid relational + JSONB per the chosen data model), configuration loading, the Fastify server with health checks, Docker Compose, and the multi-tenant auth/RLS foundation. After this phase the stack boots end-to-end with empty data and CI passes.

### Tasks

#### 1.1 — Monorepo & tooling bootstrap

**What**: Create the pnpm workspace with `core`, `api`, `worker`, `mcp`, `web` packages, shared TS config, ESLint/Prettier, and Vitest.

**Design**:
- `pnpm-workspace.yaml` lists `packages/*`.
- `tsconfig.base.json` with `strict: true`, `target: ES2023`, `module: NodeNext`, path aliases `@core/*`.
- Root scripts: `build`, `lint`, `typecheck`, `test`, `dev`.
- CI workflow (`.github/workflows/ci.yml`): install → lint → typecheck → test.

**Testing**:
- `Unit: pnpm -r build → exit 0` (smoke).
- `Unit: a placeholder add(a,b) in core → returns sum` (verifies Vitest wiring).
- `CI: lint + typecheck pass on empty packages`.

#### 1.2 — Configuration module

**What**: A typed, validated config loader in `core` reading from environment.

**Design**:
```ts
// packages/core/src/config.ts
import { z } from 'zod';
const ConfigSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(8080),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url(),
  TOKEN_ENCRYPTION_KEY: z.string().length(64), // hex, 32 bytes
  GITHUB_WEBHOOK_SECRET: z.string().optional(),
  GITLAB_WEBHOOK_SECRET: z.string().optional(),
  SLACK_BOT_TOKEN: z.string().optional(),
  SLACK_SIGNING_SECRET: z.string().optional(),
  ANTHROPIC_API_KEY: z.string().optional(),
  LLM_MODEL: z.string().default('claude-opus-4-8'),
  PUBLIC_BASE_URL: z.string().url().default('http://localhost:8080'),
});
export type Config = z.infer<typeof ConfigSchema>;
export function loadConfig(env = process.env): Config { return ConfigSchema.parse(env); }
```
- `.env.example` documents every variable.
- Missing/invalid required vars throw at boot with the offending field name.

**Testing**:
- `Unit: valid env → Config with defaults applied (PORT=8080)`.
- `Unit: missing DATABASE_URL → ZodError mentioning "DATABASE_URL"`.
- `Unit: TOKEN_ENCRYPTION_KEY of wrong length → ZodError`.

#### 1.3 — Database schema (hybrid relational + JSONB)

**What**: Drizzle schema implementing the chosen data model: relational columns for queried fields, JSONB for provider-specific/evolving data; drizzle-kit migration generated.

**Design** (representative tables; full set follows `data-model-suggestion-3.md`):
```ts
// packages/core/src/schema/org.ts
export const organisations = pgTable('organisations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  settings: jsonb('settings').$type<OrgSettings>().default({}),   // JSONB: feature flags, privacy mode
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});

export const teams = pgTable('teams', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull(),
  parentTeamId: uuid('parent_team_id'),
}, (t) => ({ uq: unique().on(t.organisationId, t.slug) }));

export const integrations = pgTable('integrations', {
  id: uuid('id').primaryKey().defaultRandom(),
  organisationId: uuid('organisation_id').notNull().references(() => organisations.id),
  provider: varchar('provider', { length: 50 }).notNull(), // github|gitlab|jira|slack|pagerduty
  externalId: varchar('external_id', { length: 255 }),
  accessTokenEnc: text('access_token_enc'),   // libsodium sealed box
  refreshTokenEnc: text('refresh_token_enc'),
  config: jsonb('config').$type<Record<string, unknown>>().default({}), // provider-specific
  status: varchar('status', { length: 20 }).notNull().default('active'),
  lastSyncedAt: timestamp('last_synced_at', { withTimezone: true }),
});

export const pullRequests = pgTable('pull_requests', {
  id: uuid('id').primaryKey().defaultRandom(),
  repositoryId: uuid('repository_id').notNull().references(() => repositories.id),
  externalId: varchar('external_id', { length: 255 }).notNull(),
  number: integer('number').notNull(),
  state: varchar('state', { length: 20 }).notNull(),  // open|merged|closed
  authorExternal: varchar('author_external', { length: 255 }),
  firstCommitAt: timestamp('first_commit_at', { withTimezone: true }),
  openedAt: timestamp('opened_at', { withTimezone: true }).notNull(),
  firstReviewAt: timestamp('first_review_at', { withTimezone: true }),
  approvedAt: timestamp('approved_at', { withTimezone: true }),
  mergedAt: timestamp('merged_at', { withTimezone: true }),
  deployedAt: timestamp('deployed_at', { withTimezone: true }),
  raw: jsonb('raw').default({}),   // JSONB: full normalised provider payload
}, (t) => ({ uq: unique().on(t.repositoryId, t.externalId),
             idx: index('pr_repo_opened_idx').on(t.repositoryId, t.openedAt) }));
```
Additional tables per data model: `users`, `team_members`, `repositories`, `commits`, `pull_request_reviews`, `deployments`, `deployment_commits`, `incidents`, `dora_snapshots`, `cycle_time_breakdowns`, `okr_periods`, `objectives`, `key_results`, `key_result_updates`, `okr_dora_links`, `survey_campaigns`, `survey_questions`, `survey_responses`, `ai_analyses`, `executive_reports`, `industry_benchmarks`. Webhook payloads land in a `raw_events(id, organisation_id, provider, event_type, payload jsonb, received_at, processed_at)` ingestion table (append-only, GIN index on `payload`).

- All tenant tables carry `organisation_id`; RLS policies enabled in 1.5.
- GIN indexes on JSONB columns used for ad-hoc filtering (`raw_events.payload`, `integrations.config`).

**Testing**:
- `Integration (Testcontainers): run migrations on fresh Postgres → all tables exist; \d pull_requests shows JSONB raw column`.
- `Integration: insert org → team(parent) → repository → PR → cascading FKs enforced (orphan PR insert fails)`.
- `Unit: drizzle schema compiles; zod model derived from PR table validates a sample row`.

#### 1.4 — Fastify server skeleton + plugins

**What**: Boot Fastify with `db`, `redis`, `rawBody`, `swagger` plugins, a `/healthz` and `/readyz` route, and structured logging.

**Design**:
- `GET /healthz` → `200 {status:"ok"}` (liveness).
- `GET /readyz` → checks DB `SELECT 1` and Redis `PING`; `503` if either fails.
- `@fastify/swagger` + `@fastify/swagger-ui` serve `/docs` and `/openapi.json` (OpenAPI 3.1).
- `rawBody` plugin captures raw body on webhook routes for HMAC.

**Testing**:
- `Integration (inject): GET /healthz → 200 {status:"ok"}`.
- `Integration (Testcontainers): /readyz with DB+Redis up → 200; with Redis stopped → 503`.
- `Integration: GET /openapi.json → valid OpenAPI 3.1 document (openapi: "3.1.0")`.

#### 1.5 — Multi-tenancy, RLS, and auth skeleton

**What**: Postgres row-level security keyed on `organisation_id`, plus API-key auth and a session JWT verifier.

**Design**:
- Migration enables RLS on tenant tables: `CREATE POLICY tenant_isolation ON teams USING (organisation_id = current_setting('app.org_id')::uuid);` The `db` plugin sets `SET LOCAL app.org_id` per request transaction from the authenticated principal.
- `api_keys(id, organisation_id, key_hash, name, last_used_at, revoked_at)`. Auth decorator resolves `x-api-key` → hashes (SHA-256) → looks up org; rejects with `401` if missing/revoked.
- JWT (RFC 7519) bearer auth for session-derived calls; `sub`=user, `org`=organisation_id.
- Standard error envelope: `{ error: { code, message, requestId } }`.

**Testing**:
- `Integration: request with valid x-api-key → org_id set; query returns only that org's teams`.
- `Integration: RLS — user from org A cannot read org B's teams (0 rows)`.
- `Integration: missing/revoked key → 401 with error.code "unauthorized"`.

---

## Phase 2: Canonical Event Model & Ingestion Pipeline

### Purpose
Build the provider-agnostic ingestion backbone: verified webhook receipt, an append-only raw-event store, and an async normalisation pipeline that maps GitHub/GitLab/Jira payloads onto a canonical event model. This decouples noisy provider payloads from metric calculation and is the foundation every metric and AI feature reads from. It also addresses the `standards.md` gap by publishing a canonical DORA/CloudEvents-style event schema.

### Tasks

#### 2.1 — Canonical event schema (CloudEvents-aligned)

**What**: Define the normalised internal event types and publish `openapi/dora-events.schema.json`.

**Design**:
```ts
// packages/core/src/events/types.ts
export type CanonicalEventType =
  | 'pr.opened' | 'pr.review_submitted' | 'pr.approved' | 'pr.merged' | 'pr.closed'
  | 'commit.pushed' | 'deployment.started' | 'deployment.finished'
  | 'incident.opened' | 'incident.resolved'
  | 'issue.created' | 'issue.transitioned';

export interface CanonicalEvent<T = unknown> {
  id: string;                 // CloudEvents id
  source: string;             // e.g. "github:org/repo"
  type: CanonicalEventType;
  provider: 'github' | 'gitlab' | 'jira' | 'pagerduty';
  organisationId: string;
  occurredAt: string;         // ISO-8601
  subject: string;            // external entity id (PR number, deploy id)
  data: T;                    // typed per event type
}
```
- JSON Schema mirrors this (Draft 2020-12), referenced from the OpenAPI spec.

**Testing**:
- `Unit: dora-events.schema.json is valid JSON Schema (ajv compiles it)`.
- `Unit: a sample pr.merged CanonicalEvent validates against the schema`.
- `Unit: event missing occurredAt → schema validation error`.

#### 2.2 — Webhook receivers with signature verification

**What**: `POST /webhooks/github`, `/webhooks/gitlab`, `/webhooks/jira` that verify signatures, persist to `raw_events`, enqueue normalisation, and return `202` fast.

**Design**:
- GitHub: verify `X-Hub-Signature-256` HMAC-SHA256 over raw body with `GITHUB_WEBHOOK_SECRET` (timing-safe compare).
- GitLab: compare `X-Gitlab-Token` against `GITLAB_WEBHOOK_SECRET`.
- Jira: validate Connect JWT (`standards.md`) — deferred secret optional in MVP, basic shared-secret accepted.
- On success: insert `raw_events` row, `bullmq.add('normalise', {rawEventId})`, return `202 {received:true}`.
- Org resolution: map delivery (repo/workspace `external_id`) → integration → organisation; unknown source → `404` (logged, not enqueued).

**Testing**:
- `Integration (mocked): GitHub PR payload with valid signature → 202, raw_events row inserted, normalise job enqueued`.
- `Integration: GitHub payload with invalid signature → 401, no row, no job`.
- `Integration: GitLab payload with bad token → 401`.
- `Integration: payload from unknown repo → 404, no job`.
- `Unit: HMAC verify helper — known body+secret → expected digest; tampered body → false`.

#### 2.3 — Normalisation processor

**What**: BullMQ `normalise` worker that reads a `raw_event`, maps it to one or more `CanonicalEvent`s, upserts domain rows (PRs, commits, deployments, incidents), and enqueues `recompute`.

**Design**:
- Per-provider mappers: `mapGithub(raw): CanonicalEvent[]`, `mapGitlab`, `mapJira`. Pure functions in `core/events`.
- Upsert logic keyed on `(repository_id, external_id)`; updates timestamp milestones (`firstReviewAt`, `approvedAt`, `mergedAt`) idempotently — re-delivery never double-counts.
- Deployment→commit association: link deployed SHA range to `deployment_commits` for lead-time tracing.
- On success set `raw_events.processed_at`; on mapper error, job retries (BullMQ, 5 attempts exp backoff) then dead-letters with the `raw_event` preserved.

**Testing**:
- `Unit: mapGithub(pull_request.opened) → [pr.opened] with correct openedAt`.
- `Unit: mapGithub(pull_request_review.submitted, state=approved) → [pr.review_submitted, pr.approved]`.
- `Integration: process two deliveries for same PR (opened then merged) → single pull_requests row with both timestamps`.
- `Integration: malformed payload → job fails, raw_event.processed_at null, lands in DLQ`.
- `Idempotency: replaying the same raw_event twice → no duplicate rows`.

#### 2.4 — Historical backfill

**What**: A `scripts/backfill.ts` + queued job that pulls historical PRs/deployments via provider REST APIs when an integration is first connected.

**Design**:
- GitHub via Octokit `pulls.list`, `repos.listDeployments` (paginated, RFC 8288 `Link` headers).
- GitLab via `@gitbeaker/node`.
- Backfill window configurable (default 90 days). Pulls page-by-page, enqueues each as a synthetic `raw_event` reusing the normalisation pipeline.
- Rate-limit aware: respects `X-RateLimit-Remaining`, sleeps until reset.

**Testing**:
- `Integration (mocked Octokit): backfill repo with 150 PRs across 2 pages → 150 pull_requests rows`.
- `Unit: rate-limit handler — remaining=0 → waits until reset timestamp`.
- `Integration: backfill is idempotent with live webhooks (same PR → upsert, no dupes)`.

---

## Phase 3: DORA Metric Engine & Cycle Time

### Purpose
Implement the heart of the product: pure, well-tested calculators for the DORA four keys and the cycle-time breakdown, plus the snapshotting pipeline that pre-computes team-period metrics for fast dashboards. After this phase the platform can answer "what are this team's DORA metrics for the last 30 days?" via API.

### Tasks

#### 3.1 — DORA calculators (pure functions)

**What**: Functions computing each DORA key from canonical domain rows for a team + period.

**Design**:
```ts
// packages/core/src/dora/calculators.ts
interface Period { start: Date; end: Date; }
interface DoraInputs {
  deployments: Deployment[];   // success deploys to production in period
  pullRequests: PullRequest[]; // merged+deployed in period
  incidents: Incident[];       // production incidents in period
}
export function deploymentFrequency(d: Deployment[], p: Period): number;  // deploys/day
export function leadTimeForChanges(prs: PullRequest[]): number;          // median seconds firstCommitAt→deployedAt
export function meanTimeToRestore(inc: Incident[]): number;              // mean seconds openedAt→resolvedAt
export function changeFailureRate(deploys: Deployment[], inc: Incident[]): number; // failures/deploys, 0..1
export function performanceTier(metric: DoraMetric, value: number): 'elite'|'high'|'medium'|'low';
```
- Lead time uses **median** (robust to outliers); document this choice.
- CFR: deployments linked to an incident (`incidents.caused_by_deployment_id`) ÷ total production deployments.
- Tier thresholds seeded from DORA 2025 benchmarks (Phase 9 benchmarks table; hardcoded fallback constants here).
- Empty-input behaviour: frequency 0; lead time / MTTR `null` (not 0) when no data — distinguishes "fast" from "no data".

**Testing**:
- `Unit: 30 deploys over 30 days → deploymentFrequency = 1.0`.
- `Unit: PRs with lead times [1h,2h,3h] → leadTimeForChanges = 7200s (median)`.
- `Unit: 2 incidents at 10m and 30m → MTTR = 1200s`.
- `Unit: 1 failed deploy of 10 → CFR = 0.1`.
- `Unit: leadTimeForChanges([]) → null`.
- `Unit: performanceTier(deployment_frequency, 1.0) → 'elite' (per benchmark)`.

#### 3.2 — Cycle time breakdown

**What**: Stage decomposition (coding / review / deploy) from PR milestone timestamps.

**Design**:
```ts
export interface CycleTime {
  codingTimeSeconds: number | null;  // firstCommitAt → openedAt
  reviewTimeSeconds: number | null;  // openedAt → approvedAt
  deployTimeSeconds: number | null;  // approvedAt → deployedAt
  totalSeconds: number | null;
}
export function cycleTimeForPR(pr: PullRequest): CycleTime;
export function aggregateCycleTime(prs: PullRequest[]): CycleTime; // median per stage
```
- Missing milestones → that stage is `null` and excluded from the median (not treated as 0).

**Testing**:
- `Unit: PR with all milestones → correct per-stage seconds`.
- `Unit: PR missing approvedAt → reviewTime & deployTime null, codingTime present`.
- `Unit: aggregate of 3 PRs → median per stage, ignoring nulls`.

#### 3.3 — Snapshot pipeline

**What**: `recompute` worker + nightly schedule that writes `dora_snapshots` and `cycle_time_breakdowns` per team/period/granularity.

**Design**:
- Triggered by (a) normalisation completion (incremental, affected team + current period) and (b) nightly full recompute for daily/weekly/monthly granularities.
- Upsert on `(team_id, period_start, period_end, granularity)`.
- Reads only domain rows (not raw events) → deterministic and replayable.

**Testing**:
- `Integration: seed 30 days of deploys for a team → run recompute → dora_snapshots row with df=1.0`.
- `Integration: re-run recompute → upserts, no duplicate snapshot`.
- `Integration: nightly schedule produces daily+weekly+monthly rows`.

#### 3.4 — Metrics API

**What**: Read endpoints for DORA + cycle time with period/granularity filters; OpenAPI-documented.

**Design**:
- `GET /v1/teams/:teamId/dora?from=&to=&granularity=weekly` → `{ metrics: DoraSnapshot[], tiers: {...} }`.
- `GET /v1/teams/:teamId/cycle-time?from=&to=` → `{ breakdown: CycleTime, trend: [...] }`.
- `GET /v1/orgs/:orgId/dora` → aggregated across teams.
- Pagination via RFC 8288 `Link` headers for large trend series.
- Responses cached in Redis (key: team+range+granularity), invalidated on snapshot upsert.
- Privacy: endpoints return team-level only; individual breakdown gated behind `?level=individual` + org setting `allowIndividualDrilldown` (default false) → `403` otherwise.

**Testing**:
- `Integration: GET /v1/teams/:id/dora with seeded snapshots → 200, correct values & tiers`.
- `Integration: individual drilldown when org setting false → 403`.
- `Integration: cache hit — second identical request served from Redis (no DB query)`.
- `Integration: response validates against OpenAPI schema`.

---

## Phase 4: OKR Authoring & Alignment

### Purpose
Add native OKR authoring linked to DORA-derived key results — the primary differentiator versus DORA-only tools (LinearB, Swarmia). After this phase teams can create objectives, define key results that auto-pull their current value from DORA snapshots, and see live progress.

### Tasks

#### 4.1 — OKR CRUD

**What**: Endpoints for periods, objectives (hierarchical), and key results.

**Design**:
- `POST /v1/okr-periods`, `POST /v1/objectives`, `POST /v1/objectives/:id/key-results`, plus GET/PATCH/DELETE.
- Objectives support `parentId` (cascading); recursive CTE for rollup.
- `key_results.metricType ∈ {manual, deployment_frequency, lead_time, mttr, change_failure_rate, cycle_time, custom_query}` with `targetValue`, `startValue`, `direction` (up/down), `unit`.
- Validation: a DORA-linked KR must reference a team that has snapshots.

**Testing**:
- `Integration: create objective with 2 key results → persisted, returned with progress=0`.
- `Integration: create nested objective tree → GET returns hierarchy`.
- `Integration: KR with invalid metricType → 422 with field error`.

#### 4.2 — Automated KR value updates from DORA

**What**: Link DORA-typed key results to live snapshot values; recompute `currentValue` and `progress` when snapshots change.

**Design**:
- `okr_dora_links` ties a KR to a team + DORA metric.
- Progress = `(current - start) / (target - start)`, clamped 0..100, sign-adjusted for `direction=down`.
- On snapshot upsert, the recompute worker also updates linked KRs and writes a `key_result_updates` row (`source='automated'`).

**Testing**:
- `Unit: progress calc — start=1/day, current=2/day, target=3/day, up → 50%`.
- `Unit: progress for direction=down (lead time) — start=10h, current=6h, target=2h → 50%`.
- `Integration: snapshot change → linked KR currentValue + progress updated + key_result_updates row written`.

#### 4.3 — OKR progress & status rollup

**What**: Objective progress aggregates child KRs and child objectives; status derived.

**Design**:
- Objective `progress` = weighted mean of KR progress (+ child objectives).
- `status`: `on_track` (≥70% of expected pace), `at_risk` (40–70%), `off_track` (<40%) where expected pace = elapsed fraction of period × 100.
- `GET /v1/objectives/:id` returns computed progress + status.

**Testing**:
- `Unit: objective with KRs at 40% and 60% → progress 50%`.
- `Unit: at 30% with 80% of period elapsed → status off_track`.
- `Integration: GET objective tree → rolled-up parent progress`.

---

## Phase 5: Slack Notifications & Digests

### Purpose
Deliver insights into the workflow surface engineers already use (the dominant pattern across Swarmia, LinearB, Plandek). Stale-PR nudges and weekly metric digests drive engagement without dashboard visits. Can be developed in parallel with the web dashboard (Phase 6) once Phase 4 lands.

### Tasks

#### 5.1 — Slack app integration & install

**What**: Slack Bolt receiver, OAuth install flow, token storage.

**Design**:
- `/slack/install` OAuth flow → store bot token (encrypted) in `integrations` (provider=slack).
- `/slack/events` endpoint verifies `X-Slack-Signature` (HMAC over timestamp+body, `SLACK_SIGNING_SECRET`), guards replay (5-min window).
- Channel mapping config in `integrations.config` (team → channel).

**Testing**:
- `Integration: Slack event with valid signature → 200; invalid → 401`.
- `Integration: replayed timestamp >5min old → rejected`.
- `Integration: OAuth callback stores encrypted bot token`.

#### 5.2 — Stale-PR nudges

**What**: Scheduled job posting nudges for PRs open/awaiting-review beyond a threshold.

**Design**:
- Schedule (hourly business-hours): query open PRs with `openedAt`/`firstReviewAt` older than `config.stalePrHours` (default 24).
- Block Kit message: PR title, age, reviewers, link. Dedupe via Redis key per PR/day.

**Testing**:
- `Integration: PR open 30h, threshold 24h → one nudge message built`.
- `Integration: same PR nudged twice in a day → second suppressed (dedupe)`.
- `Unit: Block Kit builder → valid blocks JSON`.

#### 5.3 — Weekly metric digest

**What**: Weekly per-team DORA digest posted to Slack.

**Design**:
- Schedule (Mon 09:00 team TZ): pull latest weekly snapshot + WoW delta + tier; render Block Kit summary.
- Includes top changed metric and a one-line trend note.

**Testing**:
- `Integration: team with snapshot → digest built with WoW deltas`.
- `Unit: delta formatter — +0.5 deploys/day → "↑ 0.5/day vs last week"`.
- `Integration: team with no data → digest skipped, logged`.

---

## Phase 6: Web Dashboard

### Purpose
Provide the visual surface for DORA trends, cycle-time heatmaps, and OKR boards for users who prefer dashboards over Slack. Built on the existing API; parallelisable with Phase 5.

### Tasks

#### 6.1 — App shell, auth, team switcher

**What**: Next.js App Router shell with Auth.js (OIDC + credentials), org/team context, navigation.

**Design**:
- Auth.js with OIDC provider + credentials; session JWT carries `org`.
- Server components fetch via the API using a server-side API key / session token.
- Layout: sidebar (Dashboard, OKRs, Surveys, Settings), team switcher.

**Testing**:
- `E2E (Playwright): unauthenticated → redirect to sign-in`.
- `E2E: sign in → dashboard renders for default team`.
- `Integration: team switcher changes scoped data`.

#### 6.2 — DORA dashboard & cycle-time visualisations

**What**: Trend charts for the four keys, tier badges, cycle-time stacked bars / heatmap.

**Design**:
- Recharts line charts per metric with period selector (7/30/90 days, custom).
- Cycle-time stacked bar (coding/review/deploy) + per-stage heatmap by week.
- Period-over-period delta indicators.

**Testing**:
- `E2E: dashboard loads seeded team → four metric cards with values + tiers`.
- `E2E: switch period 30→90 days → charts refetch and update`.
- `Component: tier badge maps elite→green, low→red`.

#### 6.3 — OKR board

**What**: Objective/KR board with progress bars, status colours, inline editing.

**Design**:
- Tree view of objectives → KRs; progress bars; status pills.
- Create/edit dialogs (shadcn) hitting OKR API; DORA-linked KRs show "auto" badge + live value.

**Testing**:
- `E2E: create objective + KR via UI → appears on board`.
- `E2E: DORA-linked KR shows live current value matching API`.
- `Component: progress bar renders clamped 0..100`.

---

## Phase 7: AI-Native Layer — LLM Client, Root-Cause, Executive Summaries

### Purpose
Deliver the AI-native differentiators from `research.md`: natural-language root-cause analysis of metric anomalies and board-ready executive narratives generated from DORA data. Requires the metric engine (Phase 3) and OKRs (Phase 4) as context sources.

### Tasks

#### 7.1 — LLM client abstraction with prompt caching

**What**: `LLMClient` interface + Anthropic implementation with caching of large context blocks.

**Design**:
```ts
// packages/core/src/llm/client.ts
export interface LLMMessage { role: 'user' | 'assistant'; content: string; }
export interface LLMRequest {
  system: string;
  messages: LLMMessage[];
  cacheableContext?: string;  // large metric blob → cache_control
  maxTokens?: number;
}
export interface LLMClient {
  complete(req: LLMRequest): Promise<{ text: string; usage: TokenUsage }>;
}
```
- Anthropic impl sets `cache_control: {type:'ephemeral'}` on `cacheableContext` (metric/PR data reused across questions).
- All AI outputs persisted to `ai_analyses` (type, prompt, result, model_version, context_window).

**Testing**:
- `Unit (mocked SDK): complete() sends cacheableContext as cached block`.
- `Unit: usage captured and returned`.
- `Integration: result persisted to ai_analyses`.

#### 7.2 — Metric context builder

**What**: Assemble a structured context blob (DORA snapshots, cycle-time, recent PRs, incidents, deltas) for a team + period.

**Design**:
- `buildMetricContext(teamId, period): MetricContext` → compact JSON + a human-readable summary; bounded size (top-N PRs, anomalies highlighted).

**Testing**:
- `Unit: context for team with spike → includes the spiking metric + contributing PRs`.
- `Unit: context size capped (top 20 PRs)`.

#### 7.3 — Root-cause analysis endpoint

**What**: `POST /v1/ai/root-cause` answering "why did <metric> change?" with grounded analysis.

**Design**:
- Body: `{ teamId, metric, period }`. Builds context (7.2), calls LLM with a root-cause system prompt instructing it to cite specific PRs/incidents and avoid speculation.
- System prompt skeleton: "You are an engineering delivery analyst. Using ONLY the provided metric context, explain the most likely drivers of the change in {metric}. Cite PR numbers and incidents. If data is insufficient, say so."
- Returns narrative + referenced entities; persisted.

**Testing**:
- `Integration (mocked LLM): lead-time spike context → response references the long-review PRs`.
- `Integration: insufficient data → response states limitation, no fabricated PRs`.
- `Integration: result row in ai_analyses with type root_cause`.

#### 7.4 — Executive summary generation

**What**: `POST /v1/ai/executive-summary` translating DORA + OKR progress into board-ready prose.

**Design**:
- Body: `{ orgId, okrPeriodId?, period }`. Aggregates org-level DORA + OKR rollups → business-language narrative (velocity, reliability, OKR attainment), no DORA jargon required of the reader.
- Persisted to `executive_reports` (generated_by='ai').

**Testing**:
- `Integration (mocked LLM): org context → narrative mentioning delivery + reliability trends`.
- `Integration: persisted to executive_reports`.
- `Unit: prompt includes OKR attainment when okrPeriodId given`.

---

## Phase 8: MCP Server

### Purpose
Expose engineering metrics to AI agents/assistants via the Model Context Protocol — the standout differentiator from `standards.md` (no incumbent ships one). Lets external AI tools query DORA, cycle time, open PRs, and OKR progress without bespoke integration.

### Tasks

#### 8.1 — MCP server with core tools

**What**: An MCP server (stdio + streamable HTTP) exposing read tools backed by the core/API layer.

**Design**:
- Tools (JSON Schema inputs):
  - `get_dora_metrics({ team, period })`
  - `get_cycle_time_breakdown({ repo|team, date_range })`
  - `list_open_prs_by_age({ team, min_age_hours })`
  - `get_okr_progress({ objective_id })`
- Auth: API key passed via env/header; resolves org and applies RLS.
- Tools call the same `core` functions as the REST API (single source of truth).

**Testing**:
- `Integration: MCP client lists tools → 4 tools with valid JSON Schemas`.
- `Integration: call get_dora_metrics → returns seeded snapshot values`.
- `Integration: call with invalid team → structured MCP error`.
- `Integration: API key scoping — tools only return caller's org data`.

---

## Phase 9: Should-Have Features — OKR Drafting, Surveys, Benchmarks

### Purpose
Round out the v1.1 scope from `features.md`: AI-drafted OKRs from baseline data, the developer-experience survey module, and industry benchmark comparison. These deepen the value proposition once the core platform is solid.

### Tasks

#### 9.1 — Automated OKR drafting

**What**: `POST /v1/ai/draft-okrs` proposing objectives + DORA-aligned key results calibrated to the team's baseline.

**Design**:
- Input: `{ teamId, okrPeriodId }`. Builds baseline context (current DORA + tiers), prompts LLM to propose 2–3 objectives with measurable KRs (realistic targets = next tier up).
- Returns draft (not persisted until accepted); on accept → creates objectives/KRs via Phase 4 logic.

**Testing**:
- `Integration (mocked LLM): medium-tier team → draft KR targeting high-tier thresholds`.
- `Integration: accept draft → objectives + KRs created`.
- `Unit: draft schema validated before returning`.

#### 9.2 — Developer experience survey module

**What**: Pulse surveys (5–10 questions, quarterly), anonymous responses, correlation with metrics.

**Design**:
- `POST /v1/surveys` (campaign + questions: likert_5/7, free_text), `POST /v1/surveys/:id/dispatch` (Slack/email links), `POST /v1/surveys/:id/responses` (respondent_id nullable for anonymity; min-N suppression before showing aggregates).
- Aggregates joined to DORA snapshots for correlation views.

**Testing**:
- `Integration: create campaign + 5 questions → persisted`.
- `Integration: submit anonymous response → stored with null respondent_id`.
- `Integration: aggregate with <5 responses → suppressed (privacy)`.

#### 9.3 — Industry benchmark comparison

**What**: Seed DORA benchmark tiers and expose comparison in metrics responses.

**Design**:
- Seed `industry_benchmarks` (DORA 2025 elite/high/medium/low ranges per metric).
- Metrics API enriches each metric with `tier` + percentile band.
- `GET /v1/benchmarks` lists reference data.

**Testing**:
- `Integration: seed benchmarks → GET /v1/benchmarks returns tiers`.
- `Integration: DORA response includes correct tier for a known value`.
- `Unit: percentile band lookup boundary values`.

---

## Phase 10: Enterprise Readiness — SSO, SCIM, Hardening, Packaging

### Purpose
Address the enterprise procurement gates from `standards.md` (SAML/OIDC SSO, SCIM provisioning) and OWASP API Security hardening, plus production packaging. This unlocks larger deployments and is intentionally last so it builds on a feature-complete platform.

### Tasks

#### 10.1 — SAML 2.0 + OIDC SSO and SCIM 2.0 provisioning

**What**: Enterprise SSO (SAML/OIDC) and SCIM user lifecycle.

**Design**:
- SAML 2.0 SP + OIDC RP via Auth.js providers; per-org IdP config in `integrations.config`.
- SCIM 2.0 endpoints (RFC 7643/7644): `/scim/v2/Users` (POST/PATCH/DELETE) for create/update/deactivate, bearer-token authed per org.

**Testing**:
- `Integration: SCIM POST /Users → user created in org`.
- `Integration: SCIM PATCH active=false → user deactivated, sessions revoked`.
- `Integration (mock IdP): OIDC login → session with org claim`.

#### 10.2 — OWASP API hardening & rate limiting

**What**: Address OWASP API Top 10 (BOLA, broken auth, unrestricted resource consumption).

**Design**:
- BOLA: every resource handler asserts `resource.organisationId === principal.org` (defence-in-depth beyond RLS).
- Rate limiting (`@fastify/rate-limit`, Redis store) per API key.
- Security headers (`@fastify/helmet`), request size limits, audit log of token use.

**Testing**:
- `Integration: cross-org object access by id → 404 (not 403, no existence leak)`.
- `Integration: exceed rate limit → 429 with Retry-After`.
- `Security test: webhook endpoints reject oversized bodies`.

#### 10.3 — Production packaging & docs

**What**: Hardened Docker images, Compose for prod, deployment + admin docs, seed/demo path.

**Design**:
- Multi-stage Dockerfiles (non-root, distroless where possible).
- `docker-compose.prod.yml` with healthchecks, restart policies, volume-backed Postgres.
- `scripts/seed.ts` produces a demo org + synthetic events for evaluation.
- README quickstart; OpenAPI published at `/docs`.

**Testing**:
- `E2E: docker compose up → all services healthy; seed script populates demo data; dashboard reachable`.
- `Smoke: prod images run as non-root`.
- `Integration: /openapi.json reflects all v1 routes`.

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, schema, auth/RLS)        ─── required by everything
    │
Phase 2: Canonical events & ingestion                   ─── requires Phase 1
    │
Phase 3: DORA engine & cycle time                       ─── requires Phase 2
    │
    ├── Phase 4: OKR authoring & alignment               ─── requires Phase 3
    │       │
    │       ├── Phase 5: Slack notifications & digests    ─── can parallel with Phase 6
    │       └── Phase 6: Web dashboard                    ─── can parallel with Phase 5
    │
    ├── Phase 7: AI layer (root-cause, exec summaries)   ─── requires Phase 3 (+4 for exec)
    │       │
    │       └── Phase 9: OKR drafting, surveys, benchmarks ─ requires Phase 4 + 7
    │
    └── Phase 8: MCP server                              ─── requires Phase 3 (+4 for OKR tool)

Phase 10: Enterprise readiness (SSO/SCIM, hardening)    ─── requires a feature-complete platform (1–9)
```

**Parallelism opportunities:**
- After Phase 4: Phases 5 (Slack) and 6 (Web dashboard) can be built concurrently.
- After Phase 3: Phase 7 (AI) and Phase 8 (MCP) can proceed in parallel with the Phase 4→5/6 track.
- Phase 9 sub-tasks (surveys vs. benchmarks) are independent of each other.

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase implemented.
2. All unit and integration tests pass; new behaviour has both happy-path and edge-case tests.
3. `pnpm lint` and `pnpm typecheck` pass with no errors.
4. Testcontainers-backed integration tests pass against real Postgres + Redis.
5. `docker compose up` brings the affected services up healthy.
6. The phase's capability works end-to-end (verified via API `inject` tests, MCP client, or Playwright as appropriate).
7. New config/env vars added to `.env.example` and documented.
8. New API endpoints appear in the generated OpenAPI 3.1 document at `/openapi.json`.
9. Drizzle migrations generated, checked in, and reversible where feasible.
10. Multi-tenant isolation verified (RLS + handler-level org checks) for any new tenant data.
11. Privacy posture upheld: new metrics default to team-level; individual data gated behind opt-in.
```
