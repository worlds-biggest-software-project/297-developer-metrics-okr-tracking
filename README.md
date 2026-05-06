# Developer Metrics & OKR Tracking

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for engineering KPIs, DORA metrics, and OKR alignment that brings business-level reporting within reach of teams below enterprise scale.

Developer Metrics & OKR Tracking is an engineering intelligence platform that collects DORA four-key metrics, breaks down cycle time, and maps delivery data to objectives and key results. It is built for VPs of Engineering, engineering managers, and finance partners who need defensible answers about delivery velocity, reliability, and engineering ROI without paying enterprise SaaS prices.

---

## Why Developer Metrics & OKR Tracking?

- Enterprise platforms like Jellyfish start at $100K+/year, putting OKR-aligned engineering metrics out of reach for SMBs and growth-stage companies.
- Mid-market tools (LinearB, Swarmia) track DORA metrics well but offer limited native OKR authoring and board-level ROI reporting.
- Teams using multiple Git hosts (GitHub + GitLab) have no incumbent that cleanly unifies DORA metrics across providers.
- No incumbent answers natural-language questions like "why did our lead time spike last month?" against engineering data; reports remain manually authored.
- Predictive OKR attainment scoring from current delivery velocity is an unmet need across the category.

---

## Key Features

### DORA & Cycle Time Analytics

- DORA four-key metric collection (Deployment Frequency, Lead Time for Changes, MTTR, Change Failure Rate) from GitHub and GitLab webhooks.
- Cycle time breakdown by stage (coding time, review time, deploy time).
- Historical trend views and period-over-period comparisons.
- Team-level dashboards positioned to avoid individual surveillance concerns.

### OKR Authoring & Alignment

- OKR creation and editing with linkage to DORA-derived key results.
- Automated OKR drafting from team baseline data (planned).
- Predictive OKR attainment probability scoring based on current delivery velocity (planned).
- Executive narrative report generation translating DORA data into board-ready business language.

### Workflow & Notifications

- Jira integration for issue-to-PR linking and lead time calculation.
- Slack notifications for PR review nudges and weekly metric digests.
- Benchmark comparison against industry percentile data.

### Developer Experience

- Pulse-style developer experience survey module (5–10 questions, quarterly cadence).
- Survey responses correlated with quantitative system metrics for richer causal analysis.

---

## AI-Native Advantage

Unlike incumbents whose AI is bolted on, this project is built around AI-native interrogation: engineering leaders can ask "why did lead time spike this sprint?" and receive root-cause analysis drawing on PR, incident, and sprint data. The platform proposes well-formed engineering OKRs calibrated to the team's baseline, generates natural-language executive summaries from raw DORA data, and surfaces predictive sprint risk and bottleneck recommendations rather than waiting for managers to spot them in dashboards.

---

## Tech Stack & Deployment

The MVP ingests data via GitHub and GitLab webhooks and integrates with Jira for issue-to-deployment tracing. Slack is the primary notification surface. The DORA framework (Google), SPACE framework (Microsoft Research / ACM Queue), and DX Core 4 framework are the underlying measurement standards, all published openly. No incumbent uses GPL-encumbered components, leaving an open-source implementation legally clear.

---

## Market Context

The developer productivity insight platform market is a recognised Gartner category, estimated at $500M–$1B in 2026 with 20–25% CAGR. Incumbent pricing ranges from $149/month (CodePulse, 50 devs) and $15–20/dev/month (Swarmia) at the low end up to $100K+/year (Jellyfish) at the enterprise tier. Primary buyers are VPs of Engineering and CTOs, engineering managers running retrospectives, and CFOs seeking engineering ROI visibility.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed open-source material with a compatible licence. Copyright infringement and licence violations will not be tolerated and will result in immediate removal of the offending contribution. If you are unsure whether a piece of code, text, or other material is safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
