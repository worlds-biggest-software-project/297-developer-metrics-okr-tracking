# Standards & API Reference

> Project: Developer Metrics & OKR Tracking · Generated: 2026-05-03

## Industry Standards & Specifications

### Measurement Frameworks (De-facto Standards)

**DORA Four Keys (+ Reliability)**
- Official site: https://dora.dev/guides/dora-metrics/
- Published by Google's DevOps Research and Assessment programme. Defines the four (now five) canonical metrics for software delivery performance: Deployment Frequency, Lead Time for Changes, Mean Time to Restore (MTTR), Change Failure Rate, and Reliability (added 2023). The universal benchmark reference in this product category; all major competing tools report alignment with DORA. The framework is free to use and implement; research papers are published under academic open access.

**SPACE Framework**
- Reference paper: https://queue.acm.org/detail.cfm?id=3454124 (ACM Queue, 2021)
- Developed by researchers from GitHub, Microsoft, and the University of Victoria. Defines five productivity dimensions: Satisfaction and well-being, Performance, Activity, Communication and collaboration, Efficiency and flow. Adopted by approximately 14% of engineering teams in 2026 (DORA metrics lead at 40%). Provides a multidimensional lens to prevent over-reliance on throughput metrics alone.

**Flow Framework (Mik Kersten)**
- Reference book: *Project to Product* (2018, IT Revolution Press)
- Business-level value stream framework mapping engineering work to Flow Items (Features, Defects, Risks, Debts) and measuring Flow Velocity, Efficiency, Load, and Time. Provides vocabulary for connecting engineering throughput to business outcomes.

**DX Core 4**
- Official site: https://getdx.com/dx-core-4/
- Announced December 2024 by GetDX (Abi Noda, Laura Tacho) with advisory input from DORA, SPACE, and DevEx researchers (Nicole Forsgren, Margaret-Anne Storey, Thomas Zimmerman). Unifies prior frameworks into four dimensions — Speed, Effectiveness, Quality, and Business Impact — each with one key metric and three secondary metrics. Validated with 300+ organisations. Not an ISO standard but rapidly becoming a de-facto synthesis framework.

---

### ISO Standards

**ISO/IEC 25010:2023 — Systems and Software Quality Requirements and Evaluation (SQuaRE)**
- URL: https://www.iso.org/standard/35733.html
- Defines the software quality model with eight main characteristics: Functional Suitability, Performance Efficiency, Compatibility, Usability, Reliability, Security, Maintainability, and Portability. Relevant for defining quality dimensions captured in engineering metrics platforms and for benchmarking code quality dimensions.

**ISO/IEC 25023:2016 — Measurement of System and Software Product Quality**
- URL: https://www.iso.org/standard/35747.html
- Specifies quality measures for quantitatively evaluating system and software product quality against the characteristics defined in ISO/IEC 25010. Provides a normative basis for defining what gets measured in an engineering intelligence platform.

**ISO/IEC 33001 — Concepts and Terminology for Process Assessment**
- Provides the conceptual framework underpinning software process measurement, relevant when positioning engineering metrics against process maturity models (CMMI, SPICE).

---

### W3C & IETF Standards

**RFC 7231 — Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- The foundational REST API standard governing HTTP method semantics (GET, POST, PUT, DELETE), status codes, and content negotiation. All REST APIs in this product category use HTTP/1.1 or HTTP/2 as the transport.

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- Defines the OAuth 2.0 delegated authorisation protocol. The standard mechanism for granting engineering metrics platforms scoped read access to GitHub, GitLab, Jira, and other data sources without storing user credentials. Required for enterprise integrations.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- Defines the JWT format used in OAuth 2.0 and OpenID Connect flows. Used by engineering SaaS platforms for session tokens, API authentication, and webhook validation (e.g., Jira Connect apps use JWT validation).

**OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Authentication layer on top of OAuth 2.0. The standard mechanism for SSO integration with enterprise identity providers (Okta, Azure AD, Google Workspace). Required for enterprise procurement of SaaS tools in this category. B2B SaaS selling to enterprises typically implements both OIDC and SAML.

**SAML 2.0**
- URL: https://docs.oasis-open.org/security/saml/v2.0/
- Security Assertion Markup Language 2.0; the enterprise SSO protocol most frequently required in procurement by large organisations. Complements OIDC for identity federation with legacy enterprise IdPs.

**SCIM 2.0 (RFC 7643 / RFC 7644)**
- URL: https://www.rfc-editor.org/rfc/rfc7643 and https://www.rfc-editor.org/rfc/rfc7644
- System for Cross-domain Identity Management. The standard protocol for automated user provisioning and deprovisioning from enterprise IdPs (Okta, Azure AD) into SaaS applications. Required alongside SAML/OIDC for enterprise accounts; handles user lifecycle (create, update, deactivate) separately from login.

**RFC 8288 — Web Linking**
- URL: https://www.rfc-editor.org/rfc/rfc8288
- Defines the `Link` header for hypermedia-style API pagination. Used by REST APIs returning paginated collections of metrics, pull requests, or events.

---

### Data Model & API Specifications

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The industry-standard contract format for REST APIs. All mature engineering metrics platforms with public APIs should publish an OpenAPI 3.1 spec. Code Climate's API generates OpenAPI-compliant JSON automatically. Enables SDK generation, documentation portals, and AI tool integration (OpenAPI is increasingly used as the contract for MCP tool definitions).

**JSON:API Specification**
- URL: https://jsonapi.org/
- Standardised conventions for structuring JSON REST API responses, including pagination, filtering, and inclusion of related resources. Code Climate Velocity's public API is explicitly JSON:API-compliant. Reduces API consumer friction by making response shapes predictable.

**GraphQL**
- URL: https://spec.graphql.org/
- Query language for APIs. GitLab's DORA metrics API is available via both REST and GraphQL. Relevant for platforms exposing flexible metric queries where clients select only the fields they need.

**CloudEvents Specification**
- URL: https://cloudevents.io/ (CNCF standard)
- Standardises the format of event data produced by Git providers, CI/CD systems, and deployment tools. Relevant for normalising the diverse webhook payloads ingested from GitHub, GitLab, Jira, PagerDuty, etc. into a unified event model.

**Semantic Versioning (SemVer)**
- URL: https://semver.org/
- De-facto standard for communicating API versioning intent (MAJOR.MINOR.PATCH). Engineering metrics APIs should follow SemVer for compatibility signalling.

---

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749)** — see W3C & IETF section above.

**OpenID Connect 1.0** — see W3C & IETF section above.

**SAML 2.0** — see W3C & IETF section above.

**SCIM 2.0** — see W3C & IETF section above.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/www-project-api-security/
- Defines the ten most critical API security risks. Particularly relevant: Broken Object Level Authorization (BOLA), Broken Authentication, Unrestricted Resource Consumption (rate limiting), and Security Misconfiguration. Engineering metrics APIs exposing sensitive code and team performance data must address these.

**NIST SP 800-63B — Digital Identity Guidelines**
- URL: https://pages.nist.gov/800-63-3/sp800-63b.html
- US federal guidance on authentication assurance levels. Relevant for enterprise customers in regulated industries (finance, healthcare, government) who require MFA and specific credential management practices.

**GDPR (EU Regulation 2016/679)**
- URL: https://gdpr.eu/
- Developer activity data (commits, PR timing, review participation) may constitute personal data under GDPR. Platforms serving EU customers must implement lawful basis for processing, data subject access rights, and retention policies. Individual-level metrics require particularly careful GDPR treatment.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open protocol for exposing structured data sources and tools to AI agents. An engineering metrics platform should expose an MCP server providing tools such as: `get_dora_metrics(team, period)`, `get_cycle_time_breakdown(repo, date_range)`, `list_open_prs_by_age()`, and `get_okr_progress(objective_id)`. This would allow AI assistants and agents to pull real-time engineering data for root-cause analysis and OKR narrative generation without custom integrations. MCP tool schemas are defined using JSON Schema and published as an OpenAPI-style manifest.

---

## Similar Products — Developer Documentation & APIs

### LinearB
- **Description:** Engineering analytics platform with DORA metrics, workflow automation (gitStream, WorkerB), and AI code review integration.
- **API Documentation:** https://docs.linearb.io/api-overview/
- **REST Endpoints:** `GET/POST/PUT/DELETE https://public-api.linearb.io/api/v1/services`, `POST /api/v1/deployments`, `POST /api/v2/measurements`
- **SDKs/Libraries:** No official SDK; REST API documented with test client at https://docs.linearb.io/api-test-client/
- **Developer Guide:** https://docs.linearb.io/
- **Standards:** REST/JSON; API token authentication via `x-api-key` header
- **Authentication:** API token (passed as request header)

### Swarmia
- **Description:** Developer productivity platform tracking DORA, SPACE, and developer experience survey data with Slack integration.
- **API Documentation:** https://help.swarmia.com/getting-started/integrations/data-export/export-api
- **REST Endpoints:** Deployment ingestion API; Export API returning PR metrics, DORA metrics, and investment balance data
- **SDKs/Libraries:** None publicly documented
- **Developer Guide:** https://help.swarmia.com
- **Standards:** REST/JSON
- **Authentication:** API key

### Datadog DORA Metrics
- **Description:** Observability-native DORA metric collection integrated into the Datadog monitoring platform, with no additional instrumentation required.
- **API Documentation:** https://docs.datadoghq.com/api/latest/dora-metrics/
- **REST Endpoints:** `POST /api/v2/dora/deployments`, `POST /api/v2/dora/failures`
- **SDKs/Libraries:** Datadog client libraries (Python, Go, Java, Ruby, TypeScript) via https://docs.datadoghq.com/api/latest/
- **Developer Guide:** https://www.datadoghq.com/product/platform/dora-metrics/
- **Standards:** REST/JSON; timestamps in Unix nanoseconds, milliseconds, or seconds; up to 100 user-defined tags per event
- **Authentication:** Datadog API key + Application key

### GitLab DORA Metrics API
- **Description:** Native DORA metric tracking built into GitLab for project and group-level delivery performance, available via REST and GraphQL.
- **API Documentation:** https://docs.gitlab.com/api/dora/metrics/
- **REST Endpoints:** `GET /projects/:id/dora/metrics`, `GET /groups/:id/dora/metrics` (metric, start_date, end_date, interval parameters)
- **SDKs/Libraries:** python-gitlab, go-gitlab community SDKs; official GitLab API listed at https://docs.gitlab.com/api/
- **Developer Guide:** https://docs.gitlab.com/user/analytics/dora_metrics/
- **Standards:** REST/JSON; also available via GraphQL API
- **Authentication:** OAuth 2.0 / Personal Access Token / Project Access Token; requires Reporter role minimum

### GitHub REST API (Metrics & Statistics)
- **Description:** GitHub's REST API provides repository statistics, commit activity, PR metrics, and the organisation-level Copilot Usage Metrics API for developer productivity tracking.
- **API Documentation:** https://docs.github.com/en/rest/metrics
- **REST Endpoints:** `GET /repos/{owner}/{repo}/stats/contributors`, `GET /repos/{owner}/{repo}/stats/commit_activity`, org-level metrics at `GET /orgs/{org}/metrics/summary`
- **SDKs/Libraries:** Octokit (JavaScript, Ruby, .NET); GitHub CLI; full SDK list at https://docs.github.com/en/rest/overview/libraries
- **Developer Guide:** https://docs.github.com/en/rest/guides
- **Standards:** REST/JSON; OpenAPI spec published at https://github.com/github/rest-api-description
- **Authentication:** OAuth 2.0 (GitHub App); Personal Access Token; GitHub App JWT

### Jira REST API v3
- **Description:** Atlassian's cloud REST API for Jira, providing access to issues, sprints, velocity, and custom field data used by engineering metrics tools for issue-to-PR tracing and lead time calculation.
- **API Documentation:** https://developer.atlassian.com/cloud/jira/platform/rest/v3/intro/
- **REST Endpoints:** `GET /rest/api/3/issue/{issueKey}`, `GET /rest/api/3/search` (JQL), `GET /rest/agile/1.0/board/{boardId}/sprint`
- **SDKs/Libraries:** `jira.js` (JavaScript), `jira-python` (Python), `go-jira` (Go); listed at https://developer.atlassian.com/cloud/jira/platform/
- **Developer Guide:** https://developer.atlassian.com/cloud/jira/platform/getting-started-with-connect/
- **Standards:** REST/JSON; Atlassian Document Format (ADF) for rich content; OpenAPI spec available
- **Authentication:** OAuth 2.0 (3-legged for user-context); API tokens for basic auth (deprecated for production); JWT for Atlassian Connect apps

### PagerDuty REST API
- **Description:** Incident management platform; PagerDuty data is used by DORA tools for MTTR calculation from incident creation to resolution.
- **API Documentation:** https://developer.pagerduty.com/api-reference/
- **REST Endpoints:** `GET /incidents` (with date ranges, statuses, teams); `GET /analytics/metrics`
- **SDKs/Libraries:** pdpyras (Python), PagerDuty-API-client (Ruby)
- **Developer Guide:** https://developer.pagerduty.com/docs/
- **Standards:** REST/JSON; webhooks for real-time incident events
- **Authentication:** API key; OAuth 2.0

### Code Climate API
- **Description:** Engineering intelligence platform with 60+ metrics; exposes a fully documented, JSON:API-compliant REST API.
- **API Documentation:** https://developer.codeclimate.com/
- **Standards:** REST/JSON; JSON:API specification compliant
- **Authentication:** API token

---

## Notes

**Emerging: AI-native data access via MCP.** As AI assistants are increasingly embedded in engineering workflows, there is growing expectation that engineering platforms will expose MCP servers allowing AI agents to query metrics and generate analysis on demand. No incumbent in this category currently publishes an MCP server; this represents a meaningful differentiation opportunity for a new open-source entrant.

**No unified DORA schema standard exists.** Each platform (Datadog, GitLab, LinearB, Swarmia) uses its own JSON schema for deployment events and metric responses. There is no ISO or IETF standard governing the interchange format for DORA metric data. This is a gap that an open-source tool could address by publishing a canonical JSON Schema for DORA events and encouraging adoption via the open-source community.

**GDPR compliance is non-negotiable for EU markets.** Individual-level developer activity data is personal data under GDPR. A privacy-preserving design (team-level aggregation by default, explicit opt-in for individual drill-down) is both ethically sound and reduces regulatory risk, while also being a differentiator vs. tools like Pluralsight Flow that default to individual leaderboards.

**Enterprise authentication requirements (SSO + SCIM) gate enterprise sales.** Without SAML/OIDC SSO and SCIM provisioning, enterprise procurement teams will not approve a tool for company-wide use regardless of feature quality. These are engineering requirements that must be planned for at the architecture stage, not retrofitted.
