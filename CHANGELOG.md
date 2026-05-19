# Changelog

## v1.2 — architect-grade audit + ADR system (this PR)

Five parallel architect-grade audit passes (cross-doc consistency, operational realism / command verification, triangulation vs upstream guides, steel-man counter-arguments, decision-traceability/ADR gap) found:

- **3 P0 runbook bugs** — DR commands that would fail at 3 AM
- **7 cross-doc contradictions** introduced by v1.1's new chapters
- **2 silent doctrine violations** vs `mghabin/infra-engineering-guide`
- **3 strongest counter-arguments** the doctrine hadn't engaged with

v1.2 fixes all P0 + most P1 findings, adds the missing trade studies, formalizes the ADR system, and ships the missing `decision-trees.md` sibling artifact.

### P0 — runbook commands that wouldn't have worked at 3 AM

- **Postgres failover CLI mixed `--promote-mode` and `--promote-option` values** (`docs/08`). The doc had `--promote-mode forced --promote-option planned` — `forced` isn't a valid `--promote-mode` value (`standalone` or `switchover` only). Split runbook into emergency (`standalone` + `forced`) vs planned (`switchover` + `planned`) variants per Microsoft Learn. Also added the prerequisite that `switchover` requires a pre-assigned Reader Virtual Endpoint.
- **Cloudflare LB API URL was account-scoped, not zone-scoped** (`docs/08`). Standard Cloudflare LBs are zone-scoped (`zones/{zone_id}/load_balancers/{lb_id}`); the v1.1 URL would have returned 404/403.
- **`az containerapp revision restart` was missing the required `--revision` parameter** (`docs/08`). Added a Step 5a to query the active revision name; also added the preferred Option 5a (`az containerapp update`) that forces a new revision so a new secret value is re-read at startup.
- **Postgres stop/start 7-day auto-restart side effect** (`docs/05`). Azure auto-starts stopped Flex Servers every 7 days for maintenance patches; v1.1's nightly-stop cron would have achieved $0 savings on day 8 onwards without re-stopping. Documented + added post-start `state=Ready` check.

### P0 — cross-doc architectural contradictions resolved

- **`07` internal-only origin vs `09` direct-to-origin bypass** — these were mutually exclusive (you can't grey-cloud-bypass an origin that's behind a Cloudflare Tunnel with no public IP). Resolved with an explicit security-first vs resilience-first table; recommended security-first; documented the trade-off.
- **Hyperdrive at edge vs Postgres Private Endpoint** — Workers can't reach private endpoints. Clarified: Hyperdrive applies to non-prod / public-endpoint Postgres only; in prod-private posture, Workers call backend APIs that hold connections via PgBouncer.
- **MI/no-secrets doctrine vs failover runbook updating connection-string secret** — failover now rotates `Postgres--Host` only (MI auth); legacy connection-string variant noted as discouraged-but-supported alternative.
- **Service Bus Standard cost vs Premium-required-for-Private-Endpoint** — picks table now documents the Standard → Premium tier transition at Phase 4 and the ~$675/mo per MU cost step.

### P0 — silent doctrine violations vs `infra-engineering-guide`

- **SLSA Build L3 is "must" for prod** per `infra-engineering-guide` ch06 §9. v1.1 only achieved L2 via `actions/attest-build-provenance`. Updated `docs/11` supply-chain controls table with both options + the recommendation to default to L3 via `slsa-framework/slsa-github-generator` for production artifacts, with explicit L2-exception documentation.
- **VEX statements** (Vulnerability Exploitability eXchange) required by `infra-engineering-guide` ch06 §7. Added to the controls table via OpenVEX / `openvex/vexctl`.
- **Self-hosted Hetzner runner trust model** — `infra-engineering-guide` ch03 §4.2 mandates `runner_environment=github-hosted` OIDC claim for prod deploys; self-hosted runners can't satisfy this. Added mandatory disclosure + 4 required mitigations + comparison table (GHA Larger Runners → Azure-hosted ephemeral → AWS CodeBuild → Hetzner is the right escalation order, not jump-straight-to-Hetzner).
- **Tag taxonomy mismatch with infra-guide ch10 §3** — added `data-classification` tag (required for GDPR/SOC2) + switched to kebab-case per the upstream guide's convention.
- **Burn-rate budget alerts** (1h/6h/24h windows) required by ch10 §4 — added to Phase 1 cost guardrails.
- **Kill-switch + TTL on non-prod** required by ch10 §4 — added to Phase 1.

### P1 — stale claims v1.1 missed propagating across docs

- `docs/02` still scored Cloudflare deprecation as "A+ (only expands)" — v1.1 corrected this in `04` + CHANGELOG but missed `02`. Fixed.
- `docs/03` still said Kubernetes loses on "control-plane fees + nodes" — v1.1 corrected in `06` but missed `03`. Fixed.

### P1 — head-to-head trade studies the doctrine was missing

The steel-man pass identified that the doctrine *picked* options without showing the head-to-head against the strongest alternative. v1.2 adds:

- **GitLab CI** as `03-alternatives-considered.md` chapter — GitLab is not anti-pattern; it's a deliberate alternative for shops with operational-integration / self-hostable-control-plane / org-size-50+ constraints.
- **Entra External ID for customer auth** — full trade-off table vs Auth0 / Keycloak / FusionAuth. Doctrine still defers customer auth to code, but now with explicit reasoning (blast-radius separation, portability, sovereignty decoupling) and acknowledgment that one-throat-to-choke Azure-native shops could legitimately disagree.
- **Pulumi** as IaC alternative — Bicep + OpenTofu remain the default, but Pulumi is the right pick for orgs with strong language-native (TS/Python/.NET) culture.

### P1 — "1 weekend" Phase 1 claim defended honestly

The steel-man pass flagged "Phase 1 in 1 weekend" as the highest credibility risk — a competent solo engineer can hack a prototype in a weekend, but a hardened solo foundation is 40-80h and an org-grade landing zone is 2-4 calendar weeks. Replaced with explicit operator-profile ranges + a must-have-vs-defer guidance.

### P1 — operational nits

- `docs/07` ACA AOP cert verification mechanism corrected — it's via `clientCertificateMode: "require"` on the ingress route (not "X-Forwarded-\* rules" as v1.1 imprecisely stated).
- `docs/09` Cloudflare Secondary DNS "free on all plans" softened — AXFR-out availability depends on the zone configuration / Cloudflare account team enablement.
- `docs/08` DR region examples (`eastus` / `westus2`) made jurisdiction-aware (`<primary-region>` / `<secondary-region>` placeholders) to avoid conflicting with `docs/10` EU-residency rules.
- `docs/08` SHA placeholder (`azure/login@<sha>`) filled with the real SHA.
- `docs/08` RTO/RPO tier table mapped to `infra-engineering-guide` ch08 §4 Tier 0/1/2/3 framework.
- `docs/08` operator-identity section added (CD Managed Identity + break-glass via Entra PIM).
- `docs/01` Redis Basic self-contradiction resolved — Basic non-prod / Standard prod; updated cost ladder.
- `docs/01` observability pick clarified — OTel SDK → AppInsights as OTLP receiver (not AppInsights SDK auto-instrumentation, per `infra-engineering-guide` ch05 §2).

### New artifacts

- **[`docs/decision-trees.md`](./docs/decision-trees.md)** — sibling to the `decision-trees.md` files in `dotnet-engineering-guide` and `infra-engineering-guide`. 10 trees: cloud picker / compute / state / IdP / CI runner / DR strategy / edge / multi-region trigger / IaC / SLSA level.
- **[`adrs/`](./adrs/)** — formal ADR system (MADR + Nygard reopen-trigger format). Per the decision-traceability audit, only 4 of 44 decisions had a measurable reopen trigger in narrative chapters. v1.2 ships:
  - `adrs/README.md` — index + template + how-to
  - `adrs/ADR-001-compute-platform.md` — Azure Container Apps
  - `adrs/ADR-002-primary-cloud.md` — Microsoft Azure (with explicit decay trigger on existing-skills tiebreaker)
  - `adrs/ADR-003-ci-platform.md` — GitHub Actions + OIDC

### What didn't change

- The headline recommendation (Cloudflare + Azure) — still correct for this operator profile.
- The architecture diagram and the picks-per-capability table direction.
- The phase ordering.

### What's still deferred to v1.3

- The remaining ~40 ADRs (load-bearing decisions identified in the audit but not yet formalized).
- The 12 silent-decision rationales (Trivy / cosign keyless / Syft / gitleaks / pinact / tag count / CF Registrar / Hangfire / Workbooks / PITR 7d / Cosmos 10% RU floor / Hetzner specifically). Inlined where they affect 01/05/06/11 in this PR; full ADR for each comes later.
- SLO + error-budget chapter (`infra-engineering-guide` ch09 §1 owns it).
- LAW workspace topology + retention model.

---

## v1.1 — audit-corrected

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
