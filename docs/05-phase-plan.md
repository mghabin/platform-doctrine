# 05 — Phase plan

> Concrete sequencing from solo-builder (`$0` idle) to multi-region org. Each phase is independently shippable and each one earns its keep before the next begins.

## Phase 0 — Doctrine (now)

**Goal:** make this guide the source of truth.

- This repo (`mghabin/platform-doctrine`) holds the decision.
- Cross-link from `mghabin/infra-engineering-guide` ("for which cloud, see platform-doctrine").
- Quarterly review cadence: re-score the axes in [`02-scoring-axes.md`](./02-scoring-axes.md) against the latest evidence.

**Cost:** `$0`. **Effort:** done.

## Phase 1 — Foundation

**Goal:** stand up the always-on layer. Nothing app-specific yet.

**Honest effort estimates** (replace v1.1's "1 weekend" claim — see [`v1.2 audit findings`](../CHANGELOG.md)):

| Operator profile | Realistic time | What gets cut on the short variant |
|---|---|---|
| Prototype "I want a sandbox by Monday" | ~1 weekend (16h focused) | DNSSEC, secondary DNS, hardware 2FA, Policy enforcement, audit-log streaming, AOP on origin — defer to Phase 1.5 |
| Solo founder, has done it before, going to ship customer code | ~40-80h focused | Production-grade landing zone but still solo-ops; no SOC2 evidence yet |
| Org-grade landing zone (multi-team, multi-product, audit-evidence) | **2-4 calendar weeks** | Nothing — this is the full set |

If you take the short variant, write down what you deferred + a date to come back to it. Otherwise it never happens.

### Steps (full set)

1. **Cloudflare**
   - Create account.
   - Buy primary org domain via Cloudflare Registrar (at-cost).
   - Configure zone defaults: SSL=Full (strict), security level=medium, Always Use HTTPS=on, HSTS=on, Bot Fight Mode=on.
   - **Enable DNSSEC** at the zone and verify the DS record is published at the registrar (auto-handled by Cloudflare Registrar).
   - **Configure secondary DNS** to a backup provider (e.g. AWS Route 53 at ~$0.50/zone/month). AXFR-out from Cloudflare → secondary. Resolves the DNS-chokepoint concern from [`09-concentration-risk.md`](./09-concentration-risk.md).
   - **Enforce hardware 2FA** on every team account; disable TOTP fallback.
   - Enable Cloudflare Zero Trust (free tier, ≤50 users).

2. **Azure**
   - Create tenant (if not already).
   - Apply CAF Landing Zones via the **Bicep AVM accelerator** ([aka.ms/alz/acc/bicep](https://aka.ms/alz/acc/bicep)). The older `Azure/ALZ-Bicep` "Classic" repo was removed from the official Accelerator on **2026-02-16** and will be archived in **2027-02-16** — don't start there.
   - Management group hierarchy:
     - `Platform` (shared services: DNS, identity, observability, container registry, key vaults)
     - `Workloads`
       - `Prod` (per-product subscriptions later)
       - `Non-Prod` (ci/ppe envs)
     - `Sandbox` (devs, throwaway)
     - `Decommissioned` (40-day grace before delete)
   - First subscription under `Sandbox`.

3. **CI/CD identity**
   - User-assigned MI in the `Platform` subscription with Contributor role on each workload subscription.
   - Federated Credential for `repo:org/repo:environment:ci` (and ppe, prod).
   - Configure GHA OIDC → Azure FIC (the exact pattern from EntraAuthPatterns).

4. **Cost guardrails**
   - Azure Cost Management budget alerts: `$10` / `$50` / `$100` / `$500` thresholds.
   - **Burn-rate alerts** (1h / 6h / 24h windows) per `infra-engineering-guide` ch10 §4 — Azure Cost Management forecast alerts + a custom Logic App / GHA cron that fires when 24h spend exceeds 4× rolling-7-day baseline.
   - **Kill-switch + TTL on non-prod** per `infra-engineering-guide` ch10 §4: PR-preview environments destroy after 24h; sandbox subscription gets nuked nightly via scheduled `az group delete`.
   - Azure Policy at MG level: enforce tags `cost-center`, `service`, `env`, `owner`, `data-classification` on every resource (kebab-case per `infra-engineering-guide` ch10 §3; `data-classification` is required for GDPR/SOC2 — `Public` / `Internal` / `Confidential` / `Restricted`).
   - Azure Policy: deny resources outside your approved regions — prevents accidental cross-region sprawl.

**Cost:** `$0–10`/mo (domain registration is annual; Route 53 secondary DNS is ~$0.50/zone/mo).

## Phase 2 — Platform template repo (1-2 weekends)

**Goal:** new products start with a working baseline, not a blank page.

### Repo: `<org>/cloudflare-azure-platform`

Contains:

- **Bicep modules** (generalized from EntraAuthPatterns):
  - `modules/rg.bicep` — RG with required tags
  - `modules/log-analytics.bicep` — LAW with per-tier retention (30d non-prod, 90d prod)
  - `modules/app-insights.bicep` — wired to LAW
  - `modules/key-vault.bicep` — RBAC mode, MI access
  - `modules/container-apps-environment.bicep`
  - `modules/container-app.bicep` — scale-to-zero defaults, probes, MI
  - `modules/container-app-job.bicep` — cron + event-driven
  - `modules/postgres-flexible.bicep` — Burstable B1ms for dev/ci; ACA Job (cron) calls `az postgres flexible-server stop` nightly + start-on-demand from CI
  - `modules/service-bus.bicep` — Standard tier, RBAC
  - `modules/budget.bicep` — per-RG budget with Forecasted+Actual alerts
  - `modules/cloudflare-zone.bicep` (or Terraform — Cloudflare provider isn't first-class in Bicep)

- **GHA reusable workflow** (`.github/workflows/deploy-platform-app.yml`):
  - Build container with multi-stage Dockerfile
  - Trivy scan + fail on HIGH/CRITICAL
  - cosign sign + SLSA attest
  - Push to ACR (or GHCR)
  - Deploy to ACA via FIC
  - Smoke test `/health/ready`

- **Per-language service starter templates**:
  - `templates/dotnet-api/` — `dotnet new` template for ASP.NET Core with EntraAuthPatterns-style auth (deny-by-default, OTel, named policies, Kestrel hardening, rate limiter)
  - `templates/node-worker/` — Cloudflare Worker template with Hyperdrive Postgres pool
  - `templates/nextjs-pages/` — Cloudflare Pages-deployable Next.js template

- **Convention docs** (`/conventions`):
  - Tagging
  - Naming (`{product}-{env}-{region}-{resource}`)
  - Env handling (`appsettings.{env}.json` + KV per env)
  - Secrets retrieval (MI + KV, never .env)
  - Telemetry conventions (ActivitySource per service, structured logging, RFC 7807)

**Cost:** `$0` (repo + template; no live resources). **Effort:** ~1-2 weekends.

## Phase 3 — First product onto the platform

**Goal:** validate the `$0`-idle + push-to-deploy + Cloudflare-in-front + App Insights flow on a real workload.

### Steps

1. New per-product repo. Generated from the template.
2. Lands in its own subscription under `Workloads/Non-Prod` (or `Sandbox` if still pre-product).
3. First `git push main` triggers the reusable GHA workflow → container builds → ACA deploy.
4. Cloudflare CNAME → ACA ingress.
5. Verify:
   - `/health/ready` returns 200 through Cloudflare.
   - App Insights shows the request.
   - Postgres connects via MI (no secrets in code).
   - When idle 10 minutes, ACA scales to 0. Postgres stays warm until the nightly stop-job runs (manual stop/start; storage still bills).

**Cost:** `$5–20`/mo when truly idle; `$70–120`/mo with min=1 and real traffic. **Effort:** ~1 weekend.

## Phase 4 — Multi-region (when customers span regions)

**Goal:** survive a regional outage; serve global customers with low latency.

### Steps

1. Replicate ACA environment to a second region.
2. Postgres Flexible Server: provision cross-region read replica.
3. Cloudflare Load Balancing with health checks across the two origin pools.
4. Failover playbook: documented procedure to promote the read replica to primary if region 1 fails.
5. Document the **per-region cost** so finance can see what HA actually costs.

**When to trigger:** real customers in a second region OR contractual uptime requirement that single-region can't meet.

**Cost:** ~2× single-region. Don't do it before you need it.

## Phase 5 — Multi-team governance (when you cross ~5 services)

**Goal:** prevent the platform from becoming a tangled mess as the team grows.

### Steps

1. **Stand up Backstage** as service catalog.
   - One source of truth for: which services exist, who owns them, which APIs they expose, where the docs live, what the SLOs are.
   - Auto-discovers from the platform template's conventions.
2. **Per-product subscriptions.** Split products that were sharing a subscription into their own.
3. **Per-team management groups** under `Workloads`. Apply Azure Policy assignments at the team level (region restrictions, SKU caps, tag enforcement).
4. **IAM Identity Center / Entra group → Azure RBAC** mapping. Self-serve role requests via PIM.
5. **Service Catalog products** for self-serve provisioning of common patterns (new service, new env).

**When to trigger:** when the question "who owns this?" takes >30s to answer for any service in the org.

**Cost:** Backstage hosting (~`$50/mo` on ACA) + the time investment.

## Phase 6 — Scale-out playbook (when traffic shows up)

**Goal:** absorb growth without rewrites.

The architecture doesn't change — only the dials:

- Pin `minReplicas: 1` for hot apps.
- Promote Postgres Burstable → General Purpose with HA + read replica.
- Add ElastiCache-equivalent (Azure Cache for Redis Basic) where measured cache-hits justify it.
- Cloudflare Workers AI for edge-cacheable AI inference (e.g. embeddings).
- Azure OpenAI PTU when LLM rate limits start biting.
- Migrate heaviest workload to AKS only if ACA hits a real ceiling (rare — most orgs never).

**Cost:** scales smoothly per the cost ladder in [`01-recommendation.md`](./01-recommendation.md).

## Phase 7 — Disaster recovery + business continuity (when prod has customers)

**Goal:** prove you can survive a complete primary-region loss.

### Steps

1. Document RTO/RPO targets per service.
2. Quarterly DR drill: fail over to the secondary region. Measure actual RTO.
3. Backup verification: monthly restore of Postgres backup to a scratch instance. Verify queryable.
4. Cloudflare config-as-code (Terraform provider) so a Cloudflare account compromise can be reverted from git.
5. Break-glass procedure: documented offline procedure for "Azure is completely down" — typically, restore from R2 backups to a different cloud.

**Cost:** the drills are time; the secondary-region infra is already there from Phase 4.

## Anti-phases (what NOT to do)

- **Never start with Kubernetes.** It's a Phase 8 conversation, not a Phase 0 one.
- **Never start with multi-cloud active-active.** Pick one primary; second cloud is DR only.
- **Never deploy production without `phase-4-multi-region`** if your customers are global.
- **Never skip Phase 1's tag policy** — retrofitting tags on existing resources is brutal.
- **Never skip Phase 5's service catalog** once you cross 5 services — the mess grows faster than your ability to clean it.

## Sources

- Azure Cloud Adoption Framework — [aka.ms/caf](https://learn.microsoft.com/azure/cloud-adoption-framework/)
- Backstage — [backstage.io](https://backstage.io/)
- AWS Well-Architected DR pillar (applies cross-cloud) — [docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/welcome.html)
