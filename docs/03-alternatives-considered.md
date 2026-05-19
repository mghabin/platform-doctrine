# 03 — Alternatives considered

> Honest write-ups of the candidates that lost. Why each lost on the criteria in [`02-scoring-axes.md`](./02-scoring-axes.md).

Picking the right thing is half the value of a doctrine document. **Articulating why the alternatives are wrong is the other half** — it's what stops you re-litigating the decision every six months.

## Cloudflare + AWS (close second)

**The case for it.** AWS wins on raw stability (cleanest recent outage record), operational maturity (since 2006), talent pool (largest), service deprecation (rare), and depth of control. It's the "you don't get fired for picking it" choice.

**Why it lost (against this operator).**

- **DX gap.** CDK fights you. CloudFormation is dated. IAM is genuinely the worst part of using AWS. For a solo-to-org builder shipping features, the DX cost over a year is real productivity loss.
- **Telemetry.** CloudWatch is "ok". CloudWatch Logs Insights is weaker than KQL. Most AWS shops end up overlaying Datadog or Grafana, adding a third bill.
- **Pricing surprises.** NAT Gateway, inter-AZ traffic, KMS calls, CloudFront egress — the famous surprises are well-documented but still surface monthly. Cloudflare R2 mitigates egress but doesn't eliminate the surface area.
- **Existing skills factor.** This operator has 100+ hours on Azure. Switching costs 3-6 months of productivity. **Azure-that-you-know is operationally more stable than AWS-that-you're-learning** for the first 6-12 months — your own misconfigurations cause more downtime than Azure's outages.

**When AWS is the right answer.** Fresh-start operator with no prior cloud bias, OR an existing AWS investment, OR a regulated-industry workload that needs FedRAMP High / specific GovCloud certifications.

**The exact stack you'd build (if AWS).** Cloudflare for the edge (same). Backend: **ECS Fargate + Fargate Spot for non-prod** (no EKS control-plane fee), **Aurora Serverless v2 with `min_capacity=0`** (truly `$0` idle since late 2024), **EventBridge Scheduler → ECS Run-Task for crons**, **SQS + EventBridge for async**, **Secrets Manager + KMS**, **CloudWatch + X-Ray for telemetry**, **OpenTofu for IaC**, **AWS Organizations + Control Tower for governance**. Avoid: CloudFront (Cloudflare wins), DynamoDB (Postgres wins), Step Functions (lock-in DSL), Lambda as primary compute (runtime lock-in).

---

## Cloudflare + GCP

**The case for it.** Cloud Run is the best container PaaS in any cloud — per-request billing, scale-to-zero aggressiveness, gVisor cold-start, simple DX. GCP's IAM is the simplest of the big three. Cloud SQL Postgres is solid. Vertex AI is competitive with Bedrock.

**Why it lost.**

- **Product deprecation risk.** Cloud IoT Core killed in 2023. App Engine Standard deprecation cycles. Hangouts → Meet → Chat. Datastore → Firestore migrations. Google's institutional pattern is the wrong bet for a 10-year foundation. [killedbygoogle.com](https://killedbygoogle.com/) is depressing reading.
- **Smaller talent pool.** Hiring engineers fluent in GCP is genuinely harder than AWS/Azure in most markets.
- **Smaller account-management team per customer.** Support quality degrades faster than the big two.
- **No identity bonus.** With identity in code, the bundle effect (Workspaces / Workforce Identity) doesn't help.

**When GCP is the right answer.** Greenfield solo founder optimizing for DX. Startup willing to bet on Google's continued cloud investment. Any team already deep in BigQuery for analytics.

**The exact stack you'd build (if GCP).** Cloudflare for the edge. Backend: **Cloud Run (services + jobs)**, **Cloud SQL Postgres**, **Pub/Sub**, **Memorystore Redis**, **Secret Manager**, **Cloud Trace/Logging + OpenTelemetry**, **Workload Identity Federation for GHA**, **OpenTofu**. Avoid: Firestore (proprietary API), BigQuery (proprietary SQL dialect — fine for analytics workloads, never as primary store), AppEngine Standard, Spanner.

---

## All-Cloudflare (Workers + D1 + R2 + Queues + Durable Objects)

**The case for it.** Genuinely `$0` idle for everything. One vendor to learn. Workers cold-start is `0` ms. Global from day one. Pricing is per-request and predictable. R2 zero-egress. The simplest mental model.

**Why it lost.**

- **D1 is SQLite-class.** Fine for low-write workloads, insufficient as the primary RDBMS for a multi-product org. You can't run a real ERP / accounting system / order-management on D1.
- **Workers ceilings.** 30s wall-clock, 128 MB memory (1 GB on paid). Real backends with image processing, PDF generation, large data joins, or ML inference don't fit.
- **No long-running processes.** Background jobs that need >30 minutes are a non-starter.
- **Cron via Cron Triggers is fine but limited.** No retry semantics, no DLQ for the trigger itself.

**When all-Cloudflare is the right answer.** Static-site companies, edge-heavy products, mobile-app backends with simple state, startups in the prototyping phase that don't need an RDBMS yet.

**A useful pattern that combines with the main recommendation:** use Cloudflare for *frontend + edge + object storage* and use Azure (or AWS/GCP) for *primary state + heavyweight compute*. That's exactly the recommended stack.

---

## Fly.io (with Cloudflare in front)

**The case for it.** Best DX of any 2026 option, period. `fly launch` and `fly deploy` work. Postgres included. Scale-to-zero on Fly Machines. Per-region deployment trivial. Edge-aware routing built in.

**Why it lost.**

- **Vendor risk.** Fly is a small company. Multiple rounds of funding ups and downs. Pricing changes have happened (RAM pricing reshuffle 2023). The blast-radius if Fly hits trouble is migrating dozens of services on short notice.
- **Operational depth.** Fly has had several multi-hour outages of their own (Postgres availability incidents, build cluster issues). They're transparent about it but the bench is smaller than hyperscalers.
- **Compliance ceiling.** Limited certs (SOC 2 Type II at best as of 2026). Doesn't fit regulated workloads.

**When Fly is the right answer.** Hobby projects. Side businesses with low compliance needs. As a Plan B / portability hedge — deploy one workload there yearly to keep your migration path rehearsed.

**The pattern that uses Fly.** **Don't run primary state on Fly.** Use it for stateless compute that lives close to users. Run Postgres / queues / object storage on a hyperscaler.

---

## Self-hosted on Hetzner / OVH / bare metal

**The case for it.** Absolute cheapest raw `$`. You'd save 5-10× on compute cost vs hyperscalers. Hetzner has decades of stability. Strong EU sovereignty story. Some of the most loyal users in the industry.

**Why it lost.**

- **Ops tax.** Patch management, networking, HA, load balancing, observability — every layer you'd get for free from a hyperscaler is yours to operate. For a solo-to-org builder, that's the wrong allocation of finite attention.
- **Single-DC blast radius.** The 2021 OVH Strasbourg fire took out half of OVH globally. Hetzner has similar single-DC concentration risk. Mitigation requires multi-DC architecture that you build yourself.
- **No managed services.** Postgres? You're running it. Redis? You're running it. Kubernetes? You're running it.
- **Networking is hard.** Cross-region private networking, DDoS protection, WAF — all need bolt-on solutions.

**When self-hosting is the right answer.** Cost-sensitive scale-out where compute dominates the bill (CI runners, batch jobs, ML training). Privacy-maximizer EU-only workloads. Teams with dedicated infra engineers who *want* the control.

**A useful pattern.** Run **CI build runners** on Hetzner to slash CI costs (10-100× cheaper than GitHub Actions hosted runners), with the production app still on Azure. Best of both: cheap CI + managed prod.

---

## Vercel / Netlify

**The case for it.** Best DX for Next.js / Jamstack. Preview environments per PR. Image optimization built in. Edge functions. Massive developer love.

**Why it lost.**

- **Hostile pricing past hobby.** Bandwidth overages turn into surprise bills at scale ($40/100GB on Vercel Pro is brutal). The "free until you're successful, then expensive" model.
- **Lock-in.** Vercel-specific build-time features and ISR semantics. Migrating Off Vercel is real work.
- **Focused too narrowly.** Vercel is Next-first; Netlify is Jamstack-first. Neither is a backend platform.

**When Vercel/Netlify is the right answer.** Marketing sites. Documentation sites. Pure-frontend projects where Vercel's preview environments justify the per-seat pricing.

**The recommended substitute.** **Cloudflare Pages** does most of what Vercel/Netlify do for free, with no surprise bandwidth bills (R2-style economics).

---

## Heroku / Render / Railway

**The case for it.** Even better DX than Fly.io. `git push heroku main` is still the canonical shipping experience.

**Why it lost.**

- **Vendor risk same as Fly.io but worse.** Salesforce-owned Heroku has shed features for years (free tier killed 2022). Render and Railway are smaller still.
- **Per-app pricing.** Doesn't scale to dozens of services without becoming the largest bill in the stack.
- **Compliance ceiling.** Similar to Fly.io.

**When these are the right answer.** Single-app side projects. Bootcamps and learning. Stage-1 startups that want zero infra concerns and will migrate when revenue justifies it.

---

## Multi-cloud active-active from day one

**The case for it.** Zero single-cloud failure dependency. Negotiating leverage. Best of each cloud.

**Why it lost.**

- **Multiplies operational complexity by N.** Every primitive needs an equivalent on each cloud. Every IaC module needs N implementations.
- **Hides cost.** Bills from N clouds are harder to govern.
- **Premature.** Pick *one* primary cloud for stateful workloads. Second cloud only as DR or for specific PaaS gaps.

**When multi-cloud is the right answer.** Enterprises with regulatory requirements for vendor independence. Specific PaaS-gap scenarios (e.g. you need Cloudflare Workers AND Azure OpenAI; that's already what the recommended stack does).

---

## Kubernetes from day one (AKS / EKS / GKE)

**The case for it.** Maximum portability. Industry-standard primitives. Future-proof for any scale.

**Why it lost.**

- **Premature complexity.** A solo builder running Kubernetes is paying a tax for problems they don't have yet.
- **Higher idle cost.** Control-plane fees + nodes you can't fully scale to zero.
- **Steeper learning curve** than ACA / Cloud Run / Fargate.

**When Kubernetes is the right answer.** When ACA / Cloud Run / Fargate hits a real ceiling (rare). When you have a dedicated platform team. When you're running 50+ services that justify the operational investment.

**The recommended progression.** Start on ACA. Move to AKS only when ACA cannot handle a specific workload — which most orgs never hit.

---

## Summary

| Candidate | Why it lost (1-line) |
|---|---|
| **Cloudflare + AWS** | Worst DX + no existing-skills advantage; only wins for fresh-start operators or existing AWS shops |
| **Cloudflare + GCP** | Product-kill rate disqualifies as 10-year foundation |
| **All-Cloudflare** | D1 + Workers ceilings can't support a real multi-product RDBMS workload |
| **Cloudflare + Fly.io** | Vendor risk too high for primary stateful workloads; great as Plan B |
| **Self-hosted Hetzner/OVH** | Ops tax wrong for solo-to-org builders; great for cost-sensitive batch + EU sovereignty |
| **Vercel/Netlify** | Hostile pricing past hobby + lock-in; use Cloudflare Pages instead |
| **Heroku/Render/Railway** | Per-app pricing doesn't scale + vendor risk |
| **Multi-cloud active-active from day one** | Premature; multiplies complexity by N |
| **Kubernetes from day one** | Premature complexity; start on managed PaaS |
