# 06 — Anti-patterns

> Services explicitly **avoided** in the recommended stack, and why. Avoiding the wrong thing is half the value.

A platform doctrine that only says "use X" is incomplete. The companion list — "and don't use Y, Z, W" — is what stops slow drift back into trap-shaped choices.

This isn't a "these services are bad" list. Most of them are excellent for *some* use case. They're wrong for **this stack** because they violate one or more of:

- **Lock-in beyond what the value justifies.**
- **Pricing surprises that compound at scale.**
- **Maturity gaps relative to alternatives in the same stack.**
- **DX cost greater than the capability win.**

## Azure services to avoid

### `Cosmos DB` — by default

- **Why people pick it.** Marketed as "globally distributed NoSQL" with low-latency reads. Glossy demos.
- **Why we don't.** Proprietary API (6 different ones as of 2026: NoSQL, MongoDB-compat, Cassandra-compat, Gremlin, Table, PostgreSQL-distributed-via-Citus). Pricing model (RU/s) is genuinely hard to predict — the autoscale floor pins you at 10% of max RU/s even at zero traffic (e.g. max 4000 RU/s → minimum bill ~$175/mo for nothing). Migrating off requires rewriting data access layer.
- **The exception.** Genuine global multi-write requirements where Postgres replication latency is insufficient. Measure first; assume Postgres works.
- **Substitute.** Postgres with JSONB columns covers 95% of document-DB use cases.

### `Logic Apps` / `Power Automate`

- **Why people pick it.** Drag-and-drop workflow builder; "no-code" appeal.
- **Why we don't.** Proprietary DSL. Workflows defined in clicks are uneditable in code review. Debugging is opaque. Vendor-locked.
- **Substitute.** Write workflows as code (Hangfire, Quartz.NET, custom orchestrator on ACA Jobs).

### `Azure Functions` (Consumption tier) **with Functions-specific bindings**

- **Why people pick it.** `$0` idle for event-driven code. Easy hello-world.
- **Why we don't.** Bindings are Azure-shaped (runtime lock-in). Hard to test locally without Azure Functions Core Tools. Cold-start was the historic issue on Consumption tier; **Flex Consumption (GA 2024)** materially improved cold starts, VNet integration, and always-ready instances — but the bindings lock-in remains. For org-wide consistency, prefer containers on ACA over Functions of any flavor.
- **The exception.** Glue code that's genuinely AzureEvent → 5 lines of code → another Azure resource, with no production criticality.
- **Substitute.** ACA Jobs (containers, your code, no Functions-specific bindings).

### `App Service Plans`

- **Why people pick it.** It's the original Azure PaaS; lots of tutorials use it.
- **Why we don't.** Strictly worse than ACA for new workloads: you pay per-instance even when idle, scale-to-zero requires extra work, no native multi-revision traffic splitting.
- **The exception.** Legacy ASP.NET (not Core) workloads. Even then, containerize and move to ACA.

### `Front Door` (with custom WAF rules)

- **Why people pick it.** "Microsoft's CDN/WAF, must be the right choice in Azure."
- **Why we don't.** Cloudflare is better and cheaper at every tier. Front Door has had several notable outages. Custom WAF rules in Front Door's DSL aren't portable.
- **The exception.** Hard requirement for Private Link to internal-only Azure services that Cloudflare can't reach.
- **Substitute.** Cloudflare as the global edge.

### `Service Fabric`

- **Why people pick it.** Microsoft pushed it heavily 2015-2020.
- **Why we don't.** Maintenance mode for customer workloads — Microsoft now recommends ACA (stateless) or AKS (stateful) for new projects. **Note**: Service Fabric is *not* dead; it still runs several Azure first-party control planes (Azure SQL DB, Cosmos, Event Hubs, IoT Hub, Power BI) — the April 2026 East US PIR explicitly cited it as the substrate for Azure PubSub. But the customer-facing offering is no longer where Microsoft invests.
- **Never start a new greenfield project here**; document the migration path if you inherit one.

### `API Management` (with custom policy XML)

- **Why people pick it.** Full-featured API gateway with portal, throttling, transformation.
- **Why we don't.** Proprietary XML policy language. Pricing tiers have surprising minimums. Most teams use ≤5% of features.
- **Substitute.** Cloudflare Workers as the API gateway (for public APIs) + ASP.NET Core middleware (for internal cross-cutting concerns).

### `AKS` from day one

- **Why people pick it.** "Real engineers use Kubernetes." Future-proofing.
- **Why we don't.** Premature complexity. Steeper learning curve than ACA. Most teams don't need it for years (or ever). The operational tax (cluster upgrades, node-pool sizing, networking, CSI drivers) is real even before you write any app code. (**Note**: AKS *Free* tier is `$0` control plane — only nodes bill; AKS Standard adds ~`$73/mo` for the SLA. So "control-plane cost" is opt-in, not the primary objection — operational tax is.)
- **The right time.** When ACA cannot handle a specific workload — e.g. you need custom CRDs, multi-namespace tenancy, or specific networking that ACA doesn't expose.
- **Substitute.** ACA. Migrate to AKS later if and when needed.

## AWS services to avoid (if you end up on AWS)

### `CloudFront`

- **Substitute.** Cloudflare. Better, cheaper, simpler.

### `DynamoDB` by default

- **Substitute.** Postgres + JSONB first. DynamoDB On-Demand is genuinely cost-effective and operationally boring for known-access-pattern workloads (single-table designs, IoT telemetry, session stores). The objection isn't "DynamoDB is bad"; it's **proprietary API** (won't move off AWS without rewriting data access). Use DynamoDB when you measure single-digit-ms-at-scale demands or genuine single-table-design wins; default to Postgres when in doubt.

### `Step Functions`

- **Substitute.** Workflow code in your app, in your language.

### `Amplify`

- **Why we don't.** Tries to abstract too much; ends up fighting you.
- **Substitute.** Cloudflare Pages + ECS Fargate directly.

### `Lambda as primary compute`

- **Why we don't.** Runtime model locks you to AWS (Lambda layers, specific cold-start behavior, 15-min hard limit). **Note**: SnapStart (Java/.NET), Graviton/ARM, Provisioned Concurrency, and response streaming have closed many of the historic gaps — Lambda *is* viable for production for many shapes. The objection is **workload shape + portability**: if the workload exceeds 15-min, needs custom OS access, or you want to deploy the same container on Fargate/Cloud Run/ACA later, use containers from day one.
- **The exception.** AWS-event-driven glue (e.g. S3 → process file → SQS) where the Lambda runtime is the right primitive.
- **Substitute.** ECS Fargate (containers, your code).

### `CDK` and `CloudFormation`

- **Substitute.** OpenTofu. HCL is widely-known; CDK's Python/TS abstraction adds debugging complexity.

## Cross-cloud services to avoid

### `Vercel` / `Netlify`

- **Why people pick it.** Best Next.js DX in the world.
- **Why we don't.** Pro plan includes 1 TB Fast Data Transfer/mo, then **$0.15/GB** (≈`$15/100GB`) — overage is no longer the headline objection (was ~`$0.40/GB` pre-2024). Real objections: **per-seat pricing scales painfully** ($20/user/mo Pro), **ISR/Function billing**, and **Vercel-specific build/runtime semantics** (Edge Functions, ISR) that don't port off.
- **Substitute.** Cloudflare Pages for the static + SSR; ACA / Fargate / Cloud Run for the API.

### `Heroku` / `Render` / `Railway`

- **Why people pick it.** Best `git push deploy` experience.
- **Why we don't.** Per-app pricing scales poorly. Smaller vendor risk. Compliance ceiling.
- **The exception.** Single-app side projects with zero infra concern.

### `Fly.io`

- **Why people pick it.** Best DX of any 2026 option. Genuine global deploy.
- **Why we don't as primary.** Vendor risk (small company); pricing changes have happened.
- **The right use.** Keep as a portability hedge. Deploy one workload there yearly to keep your migration path rehearsed.

### `Firebase`

- **Why people pick it.** Frontend devs love it; fast prototyping.
- **Why we don't.** Locks you to Firestore (proprietary), Firebase Auth (decent but proprietary), GCP. Pricing scales steeply. Hard to extract data later.
- **Substitute.** Cloudflare Pages + a real Postgres backend.

### `Supabase` (as the **primary** backend)

- **Why people pick it.** "Open-source Firebase" with Postgres at the core.
- **Why we don't as the primary org platform.** Smaller vendor; some scale limits documented; auth-as-tightly-coupled-to-the-DB is sometimes too clever.
- **The exception.** Side projects, prototypes, hobby apps where you want the all-in-one experience. The Postgres data itself is portable; switching costs are modest.

### `MongoDB Atlas`

- **Why people pick it.** Familiar (everyone wrote a tutorial with Mongo at some point).
- **Why we don't.** Proprietary query/aggregation API. Schemaless ≠ schema-free at scale — you still need rigor. Postgres with JSONB does 95% of the job with stronger consistency guarantees.
- **The exception.** Genuine document-DB workloads at scale (rare). Measure first.

## Anti-pattern architectures (not services, but shapes)

### Multi-cloud active-active from day one

- **Why people pick it.** "Vendor independence", "best of each cloud", regulatory.
- **Why we don't.** Multiplies operational complexity by N. Hides cost. Premature.
- **The right approach.** Pick one primary cloud for stateful workloads. Use a second cloud only for DR or specific PaaS gaps.

### Self-built service mesh (Istio / Linkerd) under 20 services

- **Why we don't.** Mesh gives you mTLS, observability, traffic splitting — all of which ACA provides natively for free.
- **The right time.** When you have a platform team and 20+ services in a true microservices architecture.

### Self-hosted observability stack (Prometheus + Grafana + Loki + Tempo)

- **Why people pick it.** "Don't pay vendor SaaS." "Full control."
- **Why we don't as default.** App Insights + LAW are free to a point and scale predictably. Self-hosting observability requires its own ops investment.
- **The right time.** When your observability bill on the managed path exceeds the cost of a dedicated platform engineer's time.

### "Just use Kubernetes for everything"

- **Why people pick it.** "Industry standard." Resume optimization.
- **Why we don't as default.** Tax on every operation. Steeper learning curve. Doesn't pay back until you have problems Kubernetes was designed for.
- **The right time.** When ACA / Cloud Run / Fargate hits a specific ceiling.

### NoSQL by default

- **Why people pick it.** "Scale." "Flexibility."
- **Why we don't.** Postgres scales further than people assume (especially with read replicas and connection pooling). Schemaless makes long-term data evolution harder, not easier. Most NoSQL DBs have proprietary APIs.
- **The right approach.** Postgres + JSONB until measured need proves Postgres can't handle it.

### Microservices from day one

- **Why people pick it.** "Future-proof", "team independence."
- **Why we don't.** Distributed systems are a *cost* you pay for organizational reasons (team boundaries, deploy independence). Pay it when the team is big enough to need it; not before. Start with a modular monolith.

## Summary table

| Avoid | Use instead | Reason |
|---|---|---|
| Cosmos DB default | Postgres + JSONB | Proprietary API + RU/s autoscale-floor trap (10% min) |
| Logic Apps | Code workflows | Proprietary DSL |
| Functions Consumption + bindings | ACA / ACA Jobs | Runtime lock-in (Flex Consumption fixed cold-start but not the lock-in) |
| App Service Plans | ACA | Pay per instance even idle |
| Front Door | Cloudflare | Better, cheaper, fewer outages |
| Service Fabric for new workloads | ACA / AKS | Customer offering in maintenance (still runs Azure first-party control planes) |
| API Management | Cloudflare Workers + middleware | Proprietary policy XML |
| AKS day-1 | ACA | Premature complexity |
| CloudFront | Cloudflare | Worse, more expensive |
| DynamoDB default | Postgres + JSONB | Proprietary API; DynamoDB On-Demand is fine when measured |
| Step Functions | App code | Lock-in DSL |
| Amplify | Pages + Fargate | Fights you |
| Lambda as default primary compute | Fargate | Workload-shape + portability (Lambda IS viable for many shapes; pick deliberately) |
| CDK / CloudFormation | OpenTofu | Debugging complexity |
| Vercel / Netlify primary | Cloudflare Pages | Per-seat pricing + ISR/Function charges + Vercel-specific build semantics |
| Heroku / Render / Railway | Cloudflare + ACA | Per-app pricing |
| Fly.io primary | ACA (Fly as hedge) | Vendor risk |
| Firebase | Cloudflare Pages + Postgres | Vendor lock-in |
| Supabase primary | ACA + Postgres Flex | Smaller vendor |
| MongoDB Atlas | Postgres + JSONB | Proprietary API |
| Multi-cloud active-active day-1 | Single primary + DR cloud | Premature |
| Self-built service mesh <20 services | ACA built-in mTLS | Premature |
| Self-hosted observability default | App Insights + LAW | Ops tax |
| Kubernetes for everything | Managed PaaS first | Ops tax |
| NoSQL default | Postgres + JSONB | Measure first |
| Microservices day-1 | Modular monolith | Distributed systems are a cost |

## Sources

- The 12-factor app (still relevant) — [12factor.net](https://12factor.net/)
- "Choose Boring Technology" — [boringtechnology.club](https://boringtechnology.club/)
- "Stop overcomplicating it" (the case against premature microservices) — [shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity](https://shopify.engineering/deconstructing-monolith-designing-software-maximizes-developer-productivity)
- DHH on managed cloud vs self-hosting trade-offs — [world.hey.com/dhh](https://world.hey.com/dhh) (the Basecamp cloud-exit posts)
