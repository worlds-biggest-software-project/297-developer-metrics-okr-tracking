# Developer Metrics & OKR Tracking — Feature & Functionality Survey

> Candidate #297 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Jellyfish | Engineering management / OKR alignment | Commercial SaaS | https://jellyfish.co |
| LinearB | Engineering analytics + workflow automation | Commercial SaaS | https://linearb.io |
| Swarmia | Developer productivity intelligence | Commercial SaaS | https://swarmia.com |
| GetDX (DX) | Developer experience + qualitative/quantitative metrics | Commercial SaaS | https://getdx.com |
| Pluralsight Flow (formerly GitPrime) | Code analytics / engineering throughput | Commercial SaaS (bundled) | https://www.pluralsight.com/flow |
| Waydev | Engineering analytics + SPACE + DORA | Commercial SaaS | https://waydev.co |
| Plandek | AI-driven software delivery intelligence | Commercial SaaS | https://plandek.com |
| Code Climate Velocity | Code and DORA metrics via Git integration | Commercial SaaS | https://codeclimate.com |
| Datadog DORA Metrics | Observability-native DORA tracking | Commercial SaaS (add-on) | https://datadoghq.com/product/platform/dora-metrics |
| GitLab Value Stream Analytics | Built-in DORA and cycle time analytics | Commercial SaaS / self-hosted | https://docs.gitlab.com/user/analytics/dora_metrics |

---

## Feature Analysis by Solution

### Jellyfish

**Core features**
- DORA four-key metrics dashboard (deployment frequency, lead time, MTTR, change failure rate)
- Investment allocation views (how engineering time is split across product areas, incidents, debt)
- OKR integration: maps Git, CI/CD, and issue tracker activity to business objectives and key results
- Team and individual contribution reporting (code, review, planning activity)
- Capacity planning and headcount modelling
- Executive-ready dashboards and narrative reports
- Integrations: GitHub, GitLab, Bitbucket, Jira, Azure DevOps, Google Sheets, Slack, Confluence

**Differentiating features**
- Business-level alignment: uniquely strong at translating engineering metrics into OKR progress language understood by CFOs and boards
- Investment reporting: visualises what percentage of budget is spent on features vs. debt vs. incidents, enabling engineering ROI conversations

**UX patterns**
- High-polish executive dashboards; more data-dense than consumer-grade tools
- Role-based views (IC, EM, VP, C-suite) with progressive disclosure of depth
- Slack alerts and weekly digests for managers

**Integration points**
- REST integrations for all major Git hosts and issue trackers
- Google Sheets export for custom reporting
- Slack notifications and digest emails
- No publicly documented open REST API for third-party access

**Known gaps**
- Very expensive; $100K+/year puts it out of reach for most SMBs and growth-stage companies
- Individual-level metrics can trigger team distrust and surveillance concerns
- Onboarding complexity; takes weeks to configure correctly with large organisations

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### LinearB

**Core features**
- DORA metrics dashboard (all four keys + cycle time breakdown by stage)
- WorkerB automation bot: automated reviewer assignment, PR labelling, stale-PR nudges
- gitStream: policy-as-code workflow automation engine for merge rules and review requirements
- Cycle time heatmaps and bottleneck detection
- Team-level benchmarking against LinearB's anonymised peer dataset (8.1M+ pull requests)
- AI code review integration on every PR

**Differentiating features**
- gitStream policy-as-code: unique engineering-workflow-as-code capability allowing review and merge policies to be version-controlled alongside code
- WorkerB automation: cuts reviewer idle time by up to 60% via automated nudges and assignments
- AI impact measurement: tracks how tools like GitHub Copilot affect cycle time and code quality at the team level

**UX patterns**
- Developer-friendly UX; Slack-native workflow with actionable notifications rather than passive dashboards
- Free tier available; progressive upgrade to paid features
- Quick time-to-value; can connect GitHub and see data within minutes

**Integration points**
- GitHub, GitLab, Bitbucket (code hosting)
- Jira, Linear (issue trackers)
- Slack (notifications and automation)
- REST API at https://docs.linearb.io; endpoints for services, deployments, and measurements (v2)
- API authentication via API token in `x-api-key` header

**Known gaps**
- OKR integration is limited; it tracks engineering metrics but does not natively map work to OKR frameworks
- Less suited to organisations requiring board-level ROI reporting without additional tooling
- gitStream requires engineering investment to configure policies effectively

**Licence / IP notes**
- Proprietary SaaS; gitStream is Apache 2.0 open source (the automation engine component)

---

### Swarmia

**Core features**
- DORA four-key metrics tracking
- SPACE framework metrics (Satisfaction, Performance, Activity, Communication, Efficiency)
- Developer experience surveys (32-question framework, quarterly cadence)
- Investment insights (how team effort maps to product themes)
- Software capitalisation reporting (R&D spend classification for accounting)
- Personal and team Slack notifications for PR reviews and CI failures

**Differentiating features**
- Anti-surveillance positioning: explicitly team-level only; no individual-level leaderboards
- Integrated developer experience surveys correlated with quantitative metrics; rare in the market
- Software capitalisation reporting for finance teams; unique among pure-play dev metrics tools

**UX patterns**
- Clean, modern UI; considered one of the most approachable in the category
- Onboarding in minutes; minimal configuration required
- Slack-first notifications rather than requiring constant dashboard visits

**Integration points**
- GitHub, GitLab (code hosting)
- Jira, Linear (issue trackers)
- Slack (notifications)
- Atlassian Marketplace listing
- Export API for DORA metrics, PR metrics, and investment balance data (https://help.swarmia.com)
- Deployment API for reporting deployments programmatically

**Known gaps**
- Smaller feature set than enterprise platforms; lacks deep OKR alignment and executive narrative reporting
- Limited Git host support (no Bitbucket or Azure DevOps)
- Survey module may be insufficient for organisations that want richer qualitative data

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### GetDX (DX)

**Core features**
- Developer experience surveys (quarterly, 5–10 questions; research-backed)
- Quantitative metrics from GitHub, GitLab, and other system integrations
- Data bridging: uses survey responses to calibrate objective system metrics
- DX AI Measurement Framework: tracks impact of AI-assisted coding tools on productivity
- Out-of-the-box and custom metrics views
- DORA metrics tracking

**Differentiating features**
- Designed by the researchers behind DORA, SPACE, and DevEx (Nicole Forsgren, Abi Noda)
- DX Core 4 framework (speed, effectiveness, quality, business impact) built into the platform
- Data bridging technique correlates subjective survey data with objective system metrics for richer causal analysis

**UX patterns**
- Research-led product design; emphasis on validity of measurement over dashboard vanity
- Benchmarking data from 300+ organisations in anonymised form
- Manager-facing insights summarising team health rather than individual scores

**Integration points**
- GitHub, GitLab (code hosting)
- Atlassian Compass integration
- Custom metrics API for user-defined metrics
- Survey distribution via email and Slack

**Known gaps**
- Pricing not publicly listed; custom enterprise pricing can be prohibitive
- Survey fatigue risk with quarterly frequency; some teams report low response rates
- Less automation-oriented than LinearB; primarily measurement rather than workflow optimisation

**Licence / IP notes**
- Proprietary SaaS; research papers underlying the frameworks are published under academic open access

---

### Pluralsight Flow (formerly GitPrime)

**Core features**
- Git-derived code analytics (commits, PR analysis, coding patterns)
- DORA metrics tracking
- Throughput and efficiency metrics (review cycle time, active days, impact score)
- Leaderboard mode ranking engineers on activity metrics
- Work log displaying team contribution patterns via interactive graphs

**Differentiating features**
- Long track record; one of the earliest code-analytics platforms (originated as GitPrime in 2016)
- Bundled with Pluralsight Skills learning platform for integrated skills + delivery data

**UX patterns**
- Interface retains legacy GitPrime-era design; considered dated vs. newer entrants
- Engineer-level views with activity breakdown per contributor
- Bundled enterprise portal requires navigating Pluralsight's broader product suite

**Integration points**
- GitHub, GitLab, Bitbucket, Azure DevOps
- Jira, Azure Boards
- AWS, Google Cloud integrations for deployment data

**Known gaps**
- Cannot be purchased standalone; must buy the full Pluralsight bundle
- Individual-level metrics and leaderboards attract significant criticism for promoting surveillance culture
- Dated UI; user adoption reportedly lower than competitors with modern interfaces
- Limited OKR or business-alignment features

**Licence / IP notes**
- Proprietary SaaS; Pluralsight acquired GitPrime in 2019

---

### Waydev

**Core features**
- DORA metrics and SPACE framework tracking
- Developer experience (DX) analysis
- Resource allocation: cost per epic/project and team effort distribution
- PR hygiene overview (linked/unlinked PRs, review patterns)
- Custom metric builder using internal metrics or external data via API
- AI agents for automatic team insights (in preview as of 2026)

**Differentiating features**
- Highly flexible custom metric builder; engineers can define their own formulas
- 200+ third-party integrations; widest integration coverage in the category
- API-first architecture with both inbound data ingestion and outbound data export

**UX patterns**
- Enterprise-oriented dashboards with deep configuration options
- Less consumer-friendly than Swarmia but more flexible for large organisations

**Integration points**
- 200+ integrations including all major Git hosts, CI/CD systems, and issue trackers
- Custom Waydev API for pulling and sending data
- Webhook-based real-time event ingestion

**Known gaps**
- Less known outside the US market
- UX is functional but not considered best-in-class
- OKR linking is limited compared to Jellyfish

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### Plandek

**Core features**
- Real-time DORA tracking with predictive analytics
- Dekka AI assistant: daily Slack/email summaries of risks, blockers, and recommended actions
- SmartView: AI-driven sprint and epic risk scoring with stall detection and recommended actions
- AI tool impact measurement (Copilot, Cursor, Devin effect on cycle time and merge frequency)
- Expanded 2026 benchmark metrics across Focus, Speed, Predictability, and Quality

**Differentiating features**
- Dekka AI assistant proactively surfaces issues rather than requiring manager dashboard visits
- SmartView predictive risk scoring at the sprint and epic level; automated bottleneck recommendations
- Strong predictability metrics; one of the few tools that goes beyond DORA into sprint forecasting

**UX patterns**
- AI-narrative-first UX; tool generates recommendations in natural language
- Slack-push model means insights arrive in existing workflows without additional logins

**Integration points**
- Full DevOps toolchain: Git hosts, CI/CD, project management, incident management
- Plandek integrations page lists supported connectors; custom API available (https://plandek.com/integrations)

**Known gaps**
- Newer entrant; smaller customer base and less social proof than Jellyfish or LinearB
- OKR module is less mature
- Custom API documentation is not fully public

**Licence / IP notes**
- Proprietary SaaS; no open-source components

---

### Code Climate Velocity

**Core features**
- 60+ engineering indicators beyond standard DORA metrics
- Git-derived metrics correlated with Jira and incident management data
- DORA four-key tracking
- Code review cycle time analysis
- JSON API-compliant REST API (https://developer.codeclimate.com)

**Differentiating features**
- Breadth of metrics (60+) combined with a fully documented, open REST API
- JSON:API specification compliance makes integration predictable and standards-based

**UX patterns**
- Developer-focused rather than executive-focused
- API-first design enables embedding metrics in other tools

**Integration points**
- GitHub (primary)
- Jira
- Incident management stack
- REST API compliant with JSON:API specification

**Known gaps**
- Smaller market presence than Jellyfish or LinearB post-2024
- Less OKR or business-alignment capability
- Limited developer experience survey functionality

**Licence / IP notes**
- Proprietary SaaS; the Code Climate open-source quality tool (code.codeclimate.com) is separate and MIT-licensed

---

### Datadog DORA Metrics

**Core features**
- DORA metrics ingested from APM, CI/CD pipelines, Git providers, and incident tools
- Deployment event API (send deployments programmatically; JSON events with custom tags)
- Correlates delivery performance with runtime system health (latency, error rate, availability)
- Native incident management integration for MTTR calculation

**Differentiating features**
- Uniquely bridges the gap between delivery metrics (DORA) and operational observability in a single platform
- No custom instrumentation required; Datadog infers DORA data from existing monitoring data streams

**UX patterns**
- Add-on capability within the Datadog platform; not a standalone product
- Suited to organisations already running Datadog for monitoring; low marginal cost to enable

**Integration points**
- REST API: POST /api/v2/dora/deployments; POST /api/v2/dora/failures (https://docs.datadoghq.com/api/latest/dora-metrics/)
- Jira integration (incident-to-issue creation, not metric import)
- Git providers: GitHub, GitLab, Bitbucket
- CI/CD pipelines: GitHub Actions, GitLab CI, CircleCI, Jenkins

**Known gaps**
- Only useful for existing Datadog customers; high vendor lock-in
- No OKR integration or developer experience survey capability
- DORA context is observability-centric, not engineering-management-centric

**Licence / IP notes**
- Proprietary SaaS; Datadog Agent is Apache 2.0 open source

---

### GitLab Value Stream Analytics

**Core features**
- Built-in DORA metrics for all four keys at project and group level
- Value stream analytics showing end-to-end cycle time from idea to production
- REST API and GraphQL API for DORA metric retrieval
- Deployment frequency and lead time tracked natively from GitLab pipelines

**Differentiating features**
- Zero additional tooling required for GitLab-native teams; metrics built into the platform
- GraphQL API provides flexible querying; REST API returns date/value pairs by metric and interval

**UX patterns**
- Embedded in GitLab UI; no separate login or integration needed
- Available on free GitLab tiers for basic metrics; advanced analytics on Premium/Ultimate

**Integration points**
- REST API: GET /projects/:id/dora/metrics and GET /groups/:id/dora/metrics (https://docs.gitlab.com/api/dora/metrics/)
- GraphQL API (complementary to REST)
- Requires Reporter role or above to access

**Known gaps**
- Only covers GitLab-hosted repositories; no multi-provider data aggregation
- No developer experience surveys or qualitative data
- No OKR integration or business-alignment reporting

**Licence / IP notes**
- GitLab is source-available (MIT Expat for Community Edition); the analytics features are in Premium/Ultimate tiers

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- DORA four-key metrics (Deployment Frequency, Lead Time for Changes, MTTR, Change Failure Rate)
- Cycle time breakdown by stage (coding, review, deploy)
- GitHub and GitLab integration
- Jira integration for issue-to-deployment tracing
- Team-level dashboards (not individual-only)
- Slack notifications for PR review nudges and CI failures
- Historical trend views and period-over-period comparisons

### Differentiating Features
- OKR alignment mapping engineering work to business objectives (Jellyfish)
- Policy-as-code workflow automation (LinearB gitStream)
- Developer experience surveys correlated with quantitative metrics (Swarmia, GetDX)
- AI-generated daily narrative insights pushed to Slack (Plandek Dekka)
- Observability-native DORA tracking from existing monitoring data (Datadog)
- Investment allocation and R&D capitalisation reporting (Jellyfish, Swarmia)
- Built-in zero-configuration metrics for existing platform users (GitLab)

### Underserved Areas / Opportunities
- Native OKR authoring: most tools track metrics but do not help teams draft well-formed OKRs from their baseline data
- Cross-platform DORA aggregation: teams using multiple Git hosts (GitHub + GitLab) have no tool that unifies metrics cleanly
- SMB-accessible OKR alignment: Jellyfish's OKR features are priced out of reach for teams below 100 engineers
- Predictive OKR attainment scoring: no tool currently predicts whether a team will hit its OKR targets based on current delivery velocity
- Natural-language metric interrogation: "why did lead time spike this sprint?" is not answered by any incumbent
- AI-generated executive summaries: translating DORA data into board-ready business narratives remains manual
- Developer well-being integration: survey data and system metrics remain siloed in most tools

### AI-Augmentation Candidates
- Root-cause analysis on metric anomalies (why did cycle time increase?)
- Automated OKR drafting from historical delivery baselines
- Predictive sprint risk scoring and delivery forecasting
- Automated reviewer assignment and PR workflow optimisation (LinearB WorkerB is early proof)
- Natural-language question answering over engineering data warehouses
- AI-generated executive narrative reports from raw DORA data

---

## Legal & IP Summary

No patents or licensing concerns were identified among the tools surveyed. All competitive platforms are proprietary SaaS products with no GPL-encumbered components that would create copyleft obligations for an independent open-source alternative. GitPrime/Pluralsight Flow holds historical brand equity, but the core analytics approaches (DORA metrics, cycle time, Git event analysis) are freely implementable. The DORA framework is published as open research by Google's DORA programme. The SPACE framework was published in ACM Queue (2021) under academic open access. The DX Core 4 framework is published openly by GetDX. LinearB's gitStream component is Apache 2.0 licensed. No concerns identified with building an open-source implementation of equivalent functionality.

---

## Recommended Feature Scope

**Must-have (MVP)**
- DORA four-key metric collection and dashboards from GitHub and GitLab webhooks
- Cycle time breakdown by stage (coding time, review time, deploy time)
- Jira integration for issue-to-PR linking and lead time calculation
- Team-level (not individual) performance views
- OKR creation and editing with linkage to DORA-derived key results
- Slack notification integration for PR nudges and weekly metric digests

**Should-have (v1.1)**
- AI-generated root-cause explanations for metric anomalies
- Automated OKR drafting from team baseline data
- Developer experience survey module (pulse-style, 5–10 questions, quarterly)
- Executive narrative report generation (natural-language DORA summary for non-technical stakeholders)
- Benchmark comparison against industry percentile data

**Nice-to-have (backlog)**
- Predictive sprint forecasting and OKR attainment probability scoring
- Investment allocation reporting and R&D capitalisation support
- Policy-as-code workflow automation (reviewer assignment, merge rules)
- Azure DevOps and Bitbucket integration
- AI tool impact measurement (Copilot, Cursor effect on metrics)
