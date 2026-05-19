# 01 — Recommendation

> **Cloudflare for the entire edge layer + Microsoft Azure for the regional backend, on portable primitives (OCI containers + Postgres + S3-API storage).**

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
                  │  Postgres Flexible       ← state (auto-pause B1ms)   │
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
| **Relational DB** | **Azure Postgres Flexible Server** (Burstable B1ms with auto-pause for dev/ci → GP for prod) | Open-source so portable; auto-pause = `$0` idle for non-prod; managed HA, read replicas, backups. |
| **DB connection pooling** | PgBouncer built into Flexible Server + Cloudflare Hyperdrive at the edge | Hyperdrive lets Workers query Postgres without holding connections. |
| **Document / NoSQL** | Postgres JSONB columns first; Cosmos DB **only** if you genuinely need global multi-write | Avoid Cosmos lock-in until measured need. |
| **Cache** | Cloudflare KV (edge) → Azure Cache for Redis Basic (regional) when needed | Pick by reader location. |
| **Async queues** | Azure Service Bus (transactions, sessions, DLQ) + Cloudflare Queues (edge fan-out) | Service Bus for backend reliability; Cloudflare Queues for edge work. |
| **Search** | Postgres FTS (free) → Azure AI Search (managed) when needed | Skip self-hosted Elasticsearch ops. |
| **Image / video serving** | Cloudflare Images + Stream | Transcoding + CDN delivery. |
| **Secrets** | Azure Key Vault per env, accessed via MI / FIC | No client secrets, ever. |
| **Observability** ⭐ | **App Insights + Log Analytics with KQL** | Azure's killer app. OTel SDKs in code → AI as backend → KQL queries + Workbooks for dashboards. |
| **Monitoring + alerts** ⭐ | Azure Monitor + Action Groups → Teams/email/webhook | Coherent UX; alert correlation works; cheap. |
| **AI / LLM** ⭐ | **Azure OpenAI Service** (GPT-N exclusives via Microsoft's OpenAI partnership; PTU for guaranteed capacity) + Cloudflare Workers AI for edge inference | Privacy-regulated AI: Azure OpenAI's data residency story is best. |
| **CI/CD** | GitHub Actions + OIDC → Azure FIC | Re-pointable to any cloud later. |
| **Container registry** | Azure Container Registry (ACR) + GHCR mirror | ACR pulls fastest from ACA; GHCR for public/portable. |
| **IaC** | **OpenTofu** (or keep Bicep until multi-cloud is real) | Bicep is fine while single-cloud; migrate when actually needed. |
| **Multi-product governance** | Azure Management Groups + Subscription-per-product-per-env + Azure Policy + Cost Management at MG level | CAF Landing Zones reference. |
| **Cost governance** | Azure Cost Management + Budgets + Advisor + Defender for Cloud (Standard tier when prod) | Budget alerts at `$10`/`$50`/`$100`/`$500`/`$1k`. |
| **Workload identity** | Managed Identity for Azure→Azure; FIC for GHA→Azure | No long-lived secrets anywhere. |
| **API gateway / BFF** | Cloudflare Workers in front of ACA (recommended) OR a BFF service in front of internal APIs | Cloudflare cheaper at scale; BFF pattern handles per-product auth + downstream routing. |
| **Internal admin access** | Cloudflare Zero Trust + Tunnels | Free for ≤50 users; replaces VPN; closes inbound ports on Azure. |
| **Transactional email** | Azure Communication Services OR Resend OR SendGrid | Commodity. |

## Cost ladder (USD/month)

| Stage | What's running | Monthly |
|---|---|---|
| Solo, building | ACA min=0, Postgres Flex auto-paused, Cloudflare free, R2 a few GB | **`$5–20`** |
| First product live, low traffic | 1 ACA app min=1, 1 worker on-demand, Postgres B1ms (`$15`), App Insights basic, Cloudflare Pro (`$25`) | **`$70–120`** |
| 2-3 products, multi-env, small team | Above × 3, Defender for Cloud Standard, ACR Basic, Service Bus Standard | **`$300–500`** |
| Real org, 10 employees, 50k MAU | Multi-region ACA, Postgres GP HA, Cloudflare Business (`$200/mo`), Redis Basic, Azure OpenAI PTU when needed | **`$1.5k–3k`** |
| Multi-region active-passive | Above × 2 regions, Postgres cross-region replica, Cloudflare LB | **`$5k–8k`** |

You never pay for tiers you haven't hit. The architecture is the same at every row.

## Addressing Azure's stability gap

Azure had larger headline outages than AWS in 2024. The plan mitigates by **not using** the Azure pieces with the worst track record:

| Concern | Mitigation in this stack |
|---|---|
| May 2024 Entra-cascade incident | Customer auth is done in code (Auth0 / Keycloak / FusionAuth), not Entra. Internal workforce auth on Entra is acceptable because workforce-only outages have smaller blast radius. |
| Regional outages (e.g. July 2024 Central US storm) | Multi-region from Phase 4 onward; Cloudflare LB does the failover. |
| Application Gateway / Front Door outages | Not in this stack — Cloudflare is the edge. |
| Cosmos pricing surprises | Not in this stack — Postgres Flexible has predictable pricing. |
| Defender false positives | Don't enable Defender for Cloud Standard tier until prod. |
| ARM API eventual consistency | Live with it; deterministic-name + idempotency patterns handle it. |

The remaining stability gap vs AWS is real but small (~15%) and is dwarfed by the productivity gain of leveraging Azure's tooling + telemetry advantages.

## Sources / further reading

- Azure Cloud Adoption Framework (CAF) — [aka.ms/caf](https://learn.microsoft.com/azure/cloud-adoption-framework/)
- Azure Container Apps reference — [learn.microsoft.com/azure/container-apps](https://learn.microsoft.com/azure/container-apps/)
- Cloudflare developer platform — [developers.cloudflare.com](https://developers.cloudflare.com/)
- Cloudflare R2 (zero egress) — [developers.cloudflare.com/r2](https://developers.cloudflare.com/r2/)
- Aurora Serverless v2 scale-to-zero (the equivalent we *don't* use, for reference) — [docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)
- Postgres Flexible Server auto-pause — [learn.microsoft.com/azure/postgresql/flexible-server/concepts-compute-storage](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-compute-storage)
- KQL reference — [learn.microsoft.com/azure/data-explorer/kusto/query](https://learn.microsoft.com/azure/data-explorer/kusto/query/)
