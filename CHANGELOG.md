# Changelog

## v1.1 — audit-corrected (this PR)

Four parallel deep-validation agents (Azure facts, non-Azure facts, pricing, strategic logic) found 20+ factual errors and 5 missing chapters. This version fixes the errors and adds the chapters.

### MUST-FIX errors corrected

**Azure factual:**

- **Postgres Flex "auto-pause" was fabricated.** Linked source URL even 404s. Replaced with the reality: manual stop/start via `az postgres flexible-server stop`, ACA-Job-cron pattern for nightly stops, storage still bills.
- **PgBouncer doesn't work on Burstable tier.** Disclosure added — PgBouncer is GP/MO only; B1ms dev/CI needs direct connections or Hyperdrive at the edge.
- **Cosmos DB has 6 APIs, not 4.** Added Table and PostgreSQL-via-Citus to the list.
- **Service Fabric isn't "effectively dead."** Microsoft's April 2026 East US incident PIR explicitly cited Service Fabric as the substrate for Azure PubSub control plane. Reframed as "maintenance mode for customer workloads; still runs Azure first-party control planes; don't start greenfield here".
- **ALZ-Bicep Classic was deprecated 2026-02-16** and is being archived 2027-02-16. Phase 1 now points at the Bicep AVM accelerator (`aka.ms/alz/acc/bicep`).
- **Azure OpenAI "first access to new GPT models"** is no longer reliably true — model availability is now roughly simultaneous with OpenAI's direct API. Reframed as: pick Azure OpenAI for VNet/Private Link/CMK/PTU/governance, not "first access".
- **2024 outage dates** ("May 2024", "July 2024") softened where exact dates couldn't be cited; recharacterized as "2024 Entra-cascade incidents" / "weather-driven Central/South Central US disruptions".

**Non-Azure factual:**

- **Workers limits were wrong both ways.** "30s wall-clock + 1 GB memory paid" → corrected to 128 MB memory (same free/paid; no memory upgrade tier), 30s CPU default on paid (5 min max), no hard wall-clock on HTTP Workers.
- **OVH 2021 fire scope:** "took out half OVH globally" → corrected to SBG2 destroyed (~10% of OVH capacity), three other Strasbourg DCs disrupted.
- **"Cloudflare only expands, never deprecates"** is false. Workers Sites was deprecated, Stream Live discontinued, Pages being folded into Workers + Static Assets. Reframed as "dramatically lower deprecation rate than GCP, not zero, with backward-compatible migration paths".
- **Cloud Run gVisor framing.** gVisor is Gen 1 only; Gen 2 microVMs have *longer* cold starts; Cloud Run Jobs use Gen 2 exclusively.
- **Vercel "$40/100GB on Pro"** is outdated by 2-3 years. Current: 1 TB/mo included, then $0.15/GB ($15/100GB). Real objections shifted to per-seat pricing + ISR/Function charges + build semantics.
- **AKS "control-plane cost"** is misleading. AKS Free tier is $0 control plane; AKS Standard adds ~$73/mo for the SLA. Real objection is operational tax, not control-plane fees. (EKS at $0.10/hr ≈ $73/mo mandatory was correct.)
- **DynamoDB / Lambda / Functions Consumption** softened — proprietary-API and runtime lock-in are the real objections; the services themselves have meaningfully improved (SnapStart, Graviton, Flex Consumption, DynamoDB On-Demand).

**Pricing / cost-ladder:**

- **Stages 4-5 split** into "infra only" vs "infra + AI". Azure OpenAI PTU isn't a "when needed" rounding error — it's a $2k-10k+/mo commitment that was hiding the real infra cost.
- **Postgres B1ms** corrected from "$15" to "~$16 24/7; $3.68/mo when stopped (storage-only)".
- **Cloudflare Pro / Business** billing cadence clarified ($20 vs $25 monthly, $200 vs $250 monthly).
- **Stealth-cost section added**: Log Analytics ingestion at 50k MAU ($35-100+/mo without sampling), Azure egress to Cloudflare beyond 100 GB free ($0.087/GB), Redis Basic has no SLA, Postgres Burstable IOPS cap.
- **Defender for Cloud** clarified — per-resource pricing, Defender for Containers can exceed $100/mo.

**Strategic logic / rubber-duck:**

- **The "~15% stability gap" number** (between Azure and AWS) was invented. Removed. Replaced with a directional statement: Azure has had headline incidents AWS hasn't, but they're mitigable by avoiding the worst-track-record Azure pieces (which the stack already does).
- **"Azure has the best telemetry in any cloud"** softened. The bundle is genuinely the strongest *cloud-native* observability stack, but Datadog/Honeycomb/Grafana on AWS or GCP is a serious peer when third-party overlays are allowed.
- **"Existing skills" weighting caveat** added prominently to both the recommendation and the scoring chapter. The factor decays sharply as the team hires.
- **Migration estimates** in the portability chapter re-baselined from "4-6 weeks small / 2-3 months multi-product" to honest ranges: toy/single-service 2-6 weeks, small real org with prod customers 3-6 months, multi-product org 6-18 months. Explicit that "weeks" is the *code* portion; the *platform* portion is months.

### New chapters added

- **[`07-origin-security-and-private-networking.md`](./docs/07-origin-security-and-private-networking.md)** — Cloudflare AOP / Tunnels, ACA VNet integration, Private Endpoints for Postgres/KV/Storage, NAT/egress, deny-by-default networking. Phased gates per [`05-phase-plan.md`](./docs/05-phase-plan.md).
- **[`08-disaster-recovery-and-backups.md`](./docs/08-disaster-recovery-and-backups.md)** — RPO/RTO per tier, full Postgres failover playbook (commands included), restore-verification GHA workflow, quarterly drill schedule.
- **[`09-concentration-risk.md`](./docs/09-concentration-risk.md)** — explicit accounting of what's on Cloudflare, secondary DNS pattern, DNSSEC + registrar lock, account-compromise runbook, "Cloudflare goes hostile" exit plan.
- **[`10-compliance-and-jurisdiction.md`](./docs/10-compliance-and-jurisdiction.md)** — GDPR + Schrems II + US CLOUD Act, data residency controls on Azure + Cloudflare, DPA/SCC/TIA paperwork, when this doctrine *doesn't* apply (EU sovereign, China, classified).
- **[`11-cicd-and-supply-chain.md`](./docs/11-cicd-and-supply-chain.md)** — GHA cost curve, when to switch to Hetzner self-hosted runners, secret rotation cadence, SBOM/SLSA/cosign keyless at scale, Dependabot grouping at 20+ services.

### What didn't change

- The headline recommendation (Cloudflare + Azure) — still correct for this operator profile, with the existing-skills caveat now explicit.
- The architecture diagram and the picks-per-capability table (only correctness fixes, not direction changes).
- The phase ordering — though Phase 1 "1 weekend" is acknowledged as solo-builder-aggressive; an org doing this properly will take longer.

## v1.0 — initial cut

Initial 6-doc structure shipped. Headline recommendation: Cloudflare + Azure. Public, MIT, CI green.
