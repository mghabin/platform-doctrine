# 01 — Recommendation

> **Cloudflare for the entire edge layer + Microsoft Azure for the regional backend, on portable primitives (OCI containers + Postgres + S3-API storage).**

> ⚠️ **Operator-specific caveat.** This recommendation reflects a scoring exercise weighted heavily on the operator's existing Azure skills (see [`02-scoring-axes.md`](./02-scoring-axes.md)). For a fresh-start operator with no prior cloud bias, Cloud Run + Cloudflare (GCP) or ECS Fargate + Cloudflare (AWS) might score higher. **Re-evaluate annually**, and especially before hiring a dedicated platform team — the existing-skills factor decays sharply as the team grows beyond the founder.

## The architecture

```
                  ┌──────────────────────────────────────────────────────┐
                  │              Cloudflare (always-on layer)             │
                  │                                                      │
                  │  Registrar · DNS · CDN · WAF · DDoS · Bot Mgmt       │
                  │  Pages         ← frontends (Next/SvelteKit/Astro/    │
                  │                  Blazor static)                       │
                  │  Workers       ← edge logic / API routing             │
                  │  R2            ← primary object storage ($0 egress)   │
                  │  KV            ← edge cache                           │
                  │  Queues        ← edge async                           │
                  │  Stream        ← video                                │
                  │  Images        ← image transforms                     │
                  │  Hyperdrive    ← Postgres connection pooler at edge   │
                  │  Zero Trust    ← internal access (replaces VPN)       │
                  └────────────────────────────┬─────────────────────────┘
                                               │ origin: regional Azure
                                               ▼
                  ┌──────────────────────────────────────────────────────┐
                  │           Azure (per-region backend)                  │
                  │                                                      │
                  │  Container Apps          ← APIs, BFFs (min=0)        │
                  │  Container Apps Jobs     ← crons + queue consumers   │
                  │  Postgres Flexible       ← state (B1ms; CI stops it) │
                  │  Service Bus             ← async, sessions, DLQ      │
                  │  Storage (Blob, Files)   ← backups, AZ-internal      │
                  │  Key Vault               ← secrets (per-env)         │
                  │  Azure OpenAI            ← LLM (GPT-N, etc.)         │
                  │  App Insights + LAW      ← observability (KQL)       │
                  │  Managed Identity / FIC  ← workload identity         │
                  │  Container Registry      ← images (or GHCR)          │
                  │  Cost Management + Budgets ← spend guardrails         │
                  └──────────────────────────────────────────────────────┘
```

Every box is `$0` or near-`$0` idle. Architecture doesn't change between hobby and prod — only `minReplicas` and SKU.

## Picks per capability

| Capability | Pick | Why |
|---|---|---|
| **DNS + registrar** | Cloudflare | At-cost registrar; seconds to propagate. |
| **CDN + WAF + DDoS + bot mgmt** | Cloudflare | Best in the world. |
| **Edge compute** | Cloudflare Workers | `$0` idle, 0 ms cold start. For: edge auth, A/B tests, redirects. |
| **Object storage — primary** | **Cloudflare R2** | S3 API + zero egress fees. |
| **Object storage — backups & Azure-internal** | Azure Blob (Cool/Archive tier) | When data must live next to the workload that produces it. |
| **Frontend hosting** | Cloudflare Pages | Free; global; SSR via Pages Functions/Workers. |
| **Backend APIs / web apps (containers)** | **Azure Container Apps** | Scale-to-zero, multi-revision, traffic-splitting, MI access to KV/Postgres, FIC for OIDC. |
| **Background workers** | ACA Jobs (event-driven via KEDA + Service Bus) | Same container as the API; Jobs are the right primitive for probe-and-exit / cron. |
| **Crons** | ACA Jobs with `triggerType: 'Schedule'` | Native cron, `$0` idle. |
| **Relational DB** | **Azure Postgres Flexible Server** — Burstable B1ms for dev/ci, GP for prod | Open-source so portable; managed HA, read replicas, backups. **Note**: Flexible Server has *manual* stop/start (NOT auto-pause like Aurora Serverless v2). For `$0` idle on non-prod, schedule a nightly `az postgres flexible-server stop` via cron/ACA Job and start-on-demand from CI. Storage continues to bill while stopped (~$3.68/mo at 32 GB). |
| **DB connection pooling** | PgBouncer built into Flexible Server (**GP/MO tiers only — not Burstable**) + Cloudflare Hyperdrive at the edge (**non-prod / public-endpoint only**) | On Burstable B1ms dev/ci, point clients directly + keep connection counts low; promote to GP for prod where PgBouncer + read replicas become available. **Hyperdrive only applies when Postgres has a public endpoint** — once prod Postgres moves to Private Endpoint per [`07-origin-security-and-private-networking.md`](./07-origin-security-and-private-networking.md), Workers cannot reach it directly. In that posture, Workers call backend APIs that hold Postgres connections via PgBouncer (no direct Worker → DB path). |
| **Document / NoSQL** | Postgres JSONB columns first; Cosmos DB **only** if you genuinely need global multi-write | Avoid Cosmos lock-in until measured need. |
| **Cache** | Cloudflare KV (edge) → Azure Cache for Redis (regional) when needed. **Basic for non-prod only (no SLA)**; **Standard tier for prod** (~$80+/mo, with SLA) | Pick by reader location and SLA need. The "Basic / Standard" split is the rule — don't run prod-critical cache on Basic. |
| **Async queues** | Azure Service Bus (transactions, sessions, DLQ) + Cloudflare Queues (edge fan-out) | Service Bus for backend reliability; Cloudflare Queues for edge work. **Tier transition**: Standard ($10/mo) is fine through Phase 3. Once you enforce Private Endpoints per [`07`](./07-origin-security-and-private-networking.md), Service Bus **Premium** is required for Private Link (adds ~$675/mo per messaging unit). Plan the cost step at Phase 4. |
| **Search** | Postgres FTS (free) → Azure AI Search (managed) when needed | Skip self-hosted Elasticsearch ops. |
| **Image / video serving** | Cloudflare Images + Stream | Transcoding + CDN delivery. |
| **Secrets** | Azure Key Vault per env, accessed via MI / FIC | No client secrets, ever. |
| **Observability** ⭐ | **App Insights + Log Analytics with KQL** (cloud-native); pivot to Datadog/Grafana Cloud only when you measurably outgrow it | The strongest *cloud-native* observability bundle. OTel SDKs → AI as backend → KQL + Workbooks for dashboards. Distributed tracing for .NET "just works". **Watch out**: at ~50k MAU with verbose logging, ingestion can hit `$50-200/mo` — enable [adaptive sampling](https://learn.microsoft.com/azure/azure-monitor/app/sampling) before traffic arrives. |
| **Monitoring + alerts** ⭐ | Azure Monitor + Action Groups → Teams/email/webhook | Coherent UX; alert correlation works; cheap. |
| **AI / LLM** ⭐ | **Azure OpenAI Service** (VNet/Private Link/CMK/Entra RBAC + PTU for guaranteed capacity) + Cloudflare Workers AI for edge inference | Privacy-regulated AI: Azure OpenAI's data residency + governance story is genuinely best. Model availability is now roughly simultaneous with OpenAI's direct API — pick Azure OpenAI for compliance/governance, not "first access". |
| **CI/CD** | GitHub Actions + OIDC → Azure FIC | Re-pointable to any cloud later. |
| **Container registry** | Azure Container Registry (ACR) + GHCR mirror | ACR pulls fastest from ACA; GHCR for public/portable. |
| **IaC** | **OpenTofu** (or keep Bicep until multi-cloud is real) | Bicep is fine while single-cloud; migrate when actually needed. |
| **Multi-product governance** | Azure Management Groups + Subscription-per-product-per-env + Azure Policy + Cost Management at MG level | CAF Landing Zones reference. |
| **Cost governance** | Azure Cost Management + Budgets + Advisor + Defender for Cloud (Standard tier when prod) | Budget alerts at `$10`/`$50`/`$100`/`$500`/`$1k`. |
| **Workload identity** | Managed Identity for Azure→Azure; FIC for GHA→Azure | No long-lived secrets anywhere. |
| **API gateway / BFF** | Cloudflare Workers in front of ACA (recommended) OR a BFF service in front of internal APIs | Cloudflare cheaper at scale; BFF pattern handles per-product auth + downstream routing. |
| **Internal admin access** | Cloudflare Zero Trust + Tunnels | Free for ≤50 users; replaces VPN; closes inbound ports on Azure. |
| **Transactional email** | Azure Communication Services OR Resend OR SendGrid | Commodity. |

## Cost ladder (USD/month, infra only — AI tokens/PTU broken out separately)

| Stage | What's running | Monthly |
|---|---|---|
| Solo, building | ACA min=0, Postgres Flex stopped overnight (storage only ~`$3.68`), Cloudflare free, R2 a few GB | **`$5–20`** |
| First product live, low traffic | 1 ACA app min=1, 1 worker on-demand, Postgres B1ms (~`$16` 24/7), App Insights basic (5 GB/mo free), Cloudflare Pro (`$20–25/mo`, annual vs monthly) | **`$50–120`** |
| 2-3 products, multi-env, small team | Above × 3 products, Defender for Cloud per-resource plans (e.g. Defender for Servers P1 ~`$5/node/mo`; **Defender for Containers can exceed `$100/mo` at scale** — review carefully before enabling org-wide), ACR Basic (`$5/mo`), Service Bus Standard (`$10/mo` base + ops) | **`$275–500`** |
| Real org, 10 employees, 50k MAU (**infra only**) | Multi-region ACA, Postgres GP HA, Cloudflare Business (`$200/mo` annual), Redis Standard (with SLA) | **`$800–1.4k`** |
| Same + meaningful AI usage (PTU committed) | Above + Azure OpenAI PTU minimum commitment | **`+$2k–10k` on top** |
| Multi-region active-passive (infra only) | Above infra × 2 regions, Postgres cross-region replica, Cloudflare LB | **`$1.6k–2.9k`** |
| Multi-region + PTU at scale | Above + PTU at scale | **`$5k–15k+`** |

You never pay for tiers you haven't hit. The architecture is the same at every row.

### Stealth costs the ladder doesn't include (model them explicitly)

| Cost | When it bites | How to mitigate |
|---|---|---|
| **Log Analytics ingestion at scale** | 50k MAU with verbose logging → 20–50 GB/mo → `$35-100+/mo` beyond the 5 GB free quota | Enable [adaptive sampling](https://learn.microsoft.com/azure/azure-monitor/app/sampling) on App Insights *before* traffic arrives |
| **Azure egress to Cloudflare** | First 100 GB/mo from Azure → public internet is free; beyond that `$0.087/GB`. At 500 GB/mo: ~`$35/mo` in Azure egress | Keep payloads small; cache aggressively at Cloudflare edge; static assets on R2 (true `$0` egress) |
| **App Insights verbose mode** | Without sampling, 50k MAU can generate 500+ GB/mo → `$1k+/mo` LAW bill | Same as above — sampling is non-optional at scale |
| **Postgres Burstable IOPS cap** | B1ms maxes at 640 IOPS; B2s at ~1.2k. IOPS-bound workloads must promote to GP D2 (+`$130/mo`) | Measure before promoting; some apps are CPU-bound, not IOPS-bound |
| **Redis Basic has no SLA** | A "real org with 10 employees" on Redis Basic is an availability risk; promote to Standard tier (`$80+/mo`) before launch | Don't run prod-criticial cache on Basic |
| **Azure OpenAI standard tokens** | Pay-per-token before PTU economics kick in: $100–2000/mo at 50k MAU | Quantify token usage *before* picking standard vs PTU |

## Addressing Azure's stability gap

Azure had larger headline outages than AWS in 2024. The plan mitigates by **not using** the Azure pieces with the worst track record:

| Concern | Mitigation in this stack |
|---|---|
| 2024 Entra-cascade incidents (worst cases cascaded into Storage SAS + App Service auth that *shouldn't* depend on Entra) | Customer auth is done in code (Auth0 / Keycloak / FusionAuth), not Entra. Internal workforce auth on Entra is acceptable because workforce-only outages have smaller blast radius. |
| Regional outages (e.g. the 2024 weather-driven Central/South Central US Azure disruptions) | Multi-region from Phase 4 onward; Cloudflare LB does the failover. |
| Application Gateway / Front Door outages | Not in this stack — Cloudflare is the edge. |
| Cosmos pricing surprises | Not in this stack — Postgres Flexible has predictable pricing. |
| Defender false positives | Don't enable Defender for Cloud Standard tier until prod. |
| ARM API eventual consistency | Live with it; deterministic-name + idempotency patterns handle it. |

The remaining stability gap vs AWS is real but modest — Azure has had headline incidents AWS hasn't (e.g. the May 2024 Entra-cascade) but they're mitigable by avoiding the Azure pieces with the worst track record (Front Door, Cosmos, Functions Consumption — all out of this stack). The gap is dwarfed by Azure's tooling + telemetry advantages, and by the productivity cost of relearning a new cloud.

## Sources / further reading

- Azure Cloud Adoption Framework (CAF) — [aka.ms/caf](https://learn.microsoft.com/azure/cloud-adoption-framework/)
- Azure Container Apps reference — [learn.microsoft.com/azure/container-apps](https://learn.microsoft.com/azure/container-apps/)
- Cloudflare developer platform — [developers.cloudflare.com](https://developers.cloudflare.com/)
- Cloudflare R2 (zero egress) — [developers.cloudflare.com/r2](https://developers.cloudflare.com/r2/)
- Aurora Serverless v2 scale-to-zero (the equivalent we *don't* use, for reference) — [docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)
- Postgres Flexible Server stop/start (manual, not auto-pause) — [learn.microsoft.com/azure/postgresql/flexible-server/how-to-stop-start-server-portal](https://learn.microsoft.com/azure/postgresql/flexible-server/how-to-stop-start-server-portal)
- KQL reference — [learn.microsoft.com/azure/data-explorer/kusto/query](https://learn.microsoft.com/azure/data-explorer/kusto/query/)
