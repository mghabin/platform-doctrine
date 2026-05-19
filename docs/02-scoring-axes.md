# 02 — Scoring axes

> The criteria and weights used to make the platform call. **Re-runnable for different operator profiles.**

## Why this chapter exists

Different operators arrive at different "right" answers from the same axes. A fresh-start solo founder, an enterprise with existing AWS investment, and a startup with Microsoft-stack muscle memory will all score the candidates differently. This chapter makes the scoring **explicit** so you can re-run it with your own weights.

## The 12 axes

| # | Axis | What it measures | Why it matters |
|---|---|---|---|
| 1 | **Stability / outage record** | Recent (last 3 years) headline outages, post-mortem quality, blast radius patterns | Determines how often you'll be paged for someone else's mistake |
| 2 | **Operational maturity** | Years in market, depth of incident-response bench, runbook quality | Determines whether the vendor has *seen* the failure mode you're about to hit |
| 3 | **Service deprecation rate** | Track record of killing products / breaking APIs | Determines whether your foundation will be there in 5 years |
| 4 | **Tooling** | CLI, SDKs, IDE integration, IaC ergonomics | Determines how fast you ship |
| 5 | **Telemetry / observability** | Built-in logging, metrics, tracing; query language; integration depth | Determines whether you can diagnose a prod incident in minutes vs hours |
| 6 | **Monitoring + alerts** | Alert correlation, channels (Teams/email/webhook), cross-account scope | Determines paging quality |
| 7 | **AI scaling** | Available models, latency, regional availability, governance | Determines AI feature velocity |
| 8 | **Control / depth of dials** | Granularity of resource configuration | Determines what you can tune when defaults bite |
| 9 | **DX** | Time from "I want to deploy X" to "X is running in prod" | Determines team productivity |
| 10 | **Pricing predictability** | Surprise-cost surface area, transparency, cost-control tooling | Determines whether finance fires you |
| 11 | **Talent pool** | Number of engineers fluent in the platform | Determines hiring cost |
| 12 | **Existing skills (operator-specific)** | Hours of muscle memory the *current operator* already has on the platform | Determines productivity in the first 6-12 months. **Decays sharply** as the team hires — new engineers bring market skill distributions, not the founder's skill bias. Weight this axis high for solo-to-3-person stage; weight near zero past 10 engineers. |

## The 2026 scoring (this guide's bias)

Scoring is a judgement, not a measurement. These reflect this author's read of the 2026 cloud landscape.

| Axis | AWS | **Azure (pick)** | GCP | Cloudflare (as full backend) |
|---|---|---|---|---|
| 1 — Stability | A+ | A | A- | A |
| 2 — Operational maturity | A+ | A | A- | A |
| 3 — Service deprecation rate | A | A | C+ (Cloud IoT Core retired Aug 2023, App Engine runtime cycles) | A (dramatically lower rate than GCP — Workers Sites and Stream Live were retired, but most expansion is additive and core primitives have backward-compatible track records) |
| 4 — Tooling | B+ | **A** (Bicep, az CLI, VS Code) | A- | A |
| 5 — Telemetry / observability (cloud-native; not counting 3rd-party overlays like Datadog/Honeycomb/Grafana Cloud) | B+ (CloudWatch + X-Ray) | **A** (App Insights + LAW + KQL — strongest cloud-native bundle; OTel + .NET tracing "just works"; Datadog-on-Azure still better at correlated dashboards) | A- (Cloud Trace/Logging + Vertex AI anomaly detection) | A- (Workers Observability) |
| 6 — Monitoring + alerts | B (dated UX) | **A** (Action Groups) | A- | A |
| 7 — AI scaling | A (Bedrock: Claude/Llama/Mistral/Nova) | **A** (Azure OpenAI: VNet/Private Link/CMK/PTU + Entra RBAC; model availability now roughly simultaneous with the direct OpenAI API — the historic "first access" gap closed in 2025/2026) | A (Vertex AI, Gemini) | A (Workers AI, edge-only) |
| 8 — Control / depth | A+ | A | A | B (constrained by edge model) |
| 9 — DX | C+ | B+ | A | A+ |
| 10 — Pricing predictability | C+ (NAT, inter-AZ) | B | B+ | A (per-request) |
| 11 — Talent pool | A+ | A | B+ | C+ |
| 12 — Existing skills (this operator) | F | **A+** | F | C |

## How the weighting collapses to "Azure"

With this operator's stated weights — **stability + tooling + telemetry + monitoring + alerts + AI scaling + control + existing skills** — Azure wins on 4 axes outright (tooling, telemetry, monitoring, alerts), ties on AI scaling, and brings a massive `12-existing-skills` bonus. Loses only on `1-stability` and `8-control` by small margins.

AWS would win this operator only if `1-stability` and `8-control` were weighted dramatically higher than the other axes, and the existing-skills factor were ignored.

## How the scoring shifts for other operator profiles

### Profile: Greenfield solo founder, no prior cloud experience

Strip the `12-existing-skills` bonus. Re-weight `9-DX` higher (because every hour matters when you're solo and learning).

**Result: GCP (Cloudflare + GCP)** — Cloud Run's DX advantage wins; the product-kill rate is a future concern, not an immediate one.

### Profile: Enterprise with existing AWS spend

`12-existing-skills` flips to AWS. `8-control` is weighted higher (enterprises need every dial). `10-pricing-predictability` is solvable via Savings Plans and Reserved Instances at their scale.

**Result: AWS (Cloudflare + AWS)** — the existing investment + control wins.

### Profile: Regulated industry (healthcare, finance, gov)

Add axis: **`13-compliance-cert-footprint`**. Weight at A++.

**Result: AWS or Azure** — both have the deepest compliance certifications (SOC 2, HIPAA, FedRAMP, ISO 27001, regional sovereignty options). GCP is weaker here.

### Profile: Startup chasing scale at any cost

`10-pricing-predictability` and `1-stability` weighted lowest. `7-AI scaling` and `4-tooling` weighted highest.

**Result: GCP (Cloud Run + Vertex AI)** — but accept the deprecation risk.

### Profile: Privacy-maximizer / EU-only

Strip the US hyperscalers from the candidate list. Add axis: **`14-data-residency`**. Weight at A++.

**Result: Hetzner + Cloudflare (EU jurisdiction)** — accept the ops tax for sovereignty.

## How to re-run for your situation

1. Take the 12 axes as a starting point. Add or drop axes that apply to *your* situation.
2. Assign weights: each axis gets `low | medium | high | critical`.
3. Score each candidate cloud A+/A/A-/B+/.../F on each axis using your read of the evidence.
4. Multiply weight × score; sum per candidate.
5. Highest sum wins.

The exact arithmetic is less important than **forcing the conversation about which axes matter**. Most bad cloud decisions come from latently weighting axes the operator didn't realize they were weighting.

## Quarterly review

These scores age. Schedule a quarterly review:

- Did the picked vendor have a headline outage? (`1-stability`)
- Did the picked vendor kill a service? (`3-service deprecation rate`)
- Did pricing change materially? (`10-pricing-predictability`)
- Did a competitor close a gap on `4`/`5`/`6`/`7`?

If three or more axes shifted by a full grade in any direction, re-run the scoring exercise.

## Sources

- Azure Cloud Adoption Framework — [aka.ms/caf](https://learn.microsoft.com/azure/cloud-adoption-framework/)
- AWS Well-Architected — [aws.amazon.com/architecture/well-architected](https://aws.amazon.com/architecture/well-architected/)
- Killed by Google — [killedbygoogle.com](https://killedbygoogle.com/)
- AWS Builder's Library (post-mortems + design notes) — [aws.amazon.com/builders-library](https://aws.amazon.com/builders-library/)
