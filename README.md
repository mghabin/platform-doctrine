# Platform Doctrine

> Opinionated, decision-mode platform-strategy for building a multi-product organization from solo-builder to enterprise.

This guide answers one question: **which cloud platform should I bet a 5-10 year organization on?**

It's a companion to:

- [`mghabin/dotnet-engineering-guide`](https://github.com/mghabin/dotnet-engineering-guide) — *how* to write the code
- [`mghabin/infra-engineering-guide`](https://github.com/mghabin/infra-engineering-guide) — *what* infra patterns to apply
- **this repo** — *which* cloud(s) to apply them on, and *why*

## TL;DR — the recommendation

> **Cloudflare for the entire edge layer + Microsoft Azure for the regional backend, on portable primitives (OCI containers + Postgres + S3-API storage) so any future migration is 2-4 weeks not 2 years.**

Full reasoning, alternatives considered, scoring matrices, cost ladders, and phase plan in [`docs/01-recommendation.md`](./docs/01-recommendation.md).

## Why a separate doctrine repo

Cloud-platform choice is a decision that costs 6-12 months to reverse and shapes everything downstream — IaC, hiring, observability, FinOps, deployment patterns. It deserves first-class documentation, not a buried section in an infra guide.

This repo also exists to be **honest about the criteria used**. The same scoring exercise can produce different recommendations for different operators — a fresh-start solo founder, an enterprise with existing AWS investment, and a startup with Microsoft-stack muscle memory will all arrive at different "right" answers from the same axes.

## What's in here

### Core (the recommendation itself)

| Doc | Purpose |
|---|---|
| [`docs/01-recommendation.md`](./docs/01-recommendation.md) | The full opinionated recommendation with architecture, picks per capability, cost ladder (with stealth-cost callouts). |
| [`docs/02-scoring-axes.md`](./docs/02-scoring-axes.md) | The criteria and weights used to make the call. Re-runnable for different operator profiles. |
| [`docs/03-alternatives-considered.md`](./docs/03-alternatives-considered.md) | Honest write-ups of the candidates that lost (AWS, GCP, all-Cloudflare, Hetzner, Fly.io). Why each lost on the criteria above. |
| [`docs/04-portability-promise.md`](./docs/04-portability-promise.md) | The explicit "if [vendor] turns hostile" test applied to every choice. What's safe; what's lock-in; what's the realistic cost to leave (months, not weeks). |
| [`docs/05-phase-plan.md`](./docs/05-phase-plan.md) | Concrete sequencing from solo-builder ($0 idle) to multi-region org. |
| [`docs/06-anti-patterns.md`](./docs/06-anti-patterns.md) | Services explicitly **avoided** in the recommended stack, and why. Avoiding the wrong thing is half the value. |

### Operational (the chapters that make this prod-real)

| Doc | Purpose |
|---|---|
| [`docs/07-origin-security-and-private-networking.md`](./docs/07-origin-security-and-private-networking.md) | Cloudflare AOP / Tunnels, ACA VNet integration, Private Endpoints, NAT/egress controls, deny-by-default network posture. |
| [`docs/08-disaster-recovery-and-backups.md`](./docs/08-disaster-recovery-and-backups.md) | RPO/RTO targets per tier, Postgres failover playbook (corrected CLI), restore-verification workflow, quarterly DR drills. |
| [`docs/09-concentration-risk.md`](./docs/09-concentration-risk.md) | What's concentrated on Cloudflare, secondary DNS, account-compromise runbook, what to do if Cloudflare goes hostile. |
| [`docs/10-compliance-and-jurisdiction.md`](./docs/10-compliance-and-jurisdiction.md) | GDPR / Schrems II / US CLOUD Act / EU Data Boundary / sanctions; when this doctrine *doesn't* apply. |
| [`docs/11-cicd-and-supply-chain.md`](./docs/11-cicd-and-supply-chain.md) | GHA OIDC + cost curve, runner trust-model, when to switch off GHA-hosted, secret rotation, SBOM/SLSA L2-vs-L3/cosign at scale. |
| [`docs/decision-trees.md`](./docs/decision-trees.md) | When the doctrine says "it depends" — 10 decision trees that return specific picks (cloud / compute / state / IdP / CI runner / DR / edge / multi-region / IaC / SLSA level). |

### Decision records

| Doc | Purpose |
|---|---|
| [`adrs/README.md`](./adrs/README.md) | Index + template for architectural decision records. Format: MADR + Nygard reopen trigger. |
| [`adrs/ADR-001-compute-platform.md`](./adrs/ADR-001-compute-platform.md) | Azure Container Apps as the default compute platform |
| [`adrs/ADR-002-primary-cloud.md`](./adrs/ADR-002-primary-cloud.md) | Microsoft Azure as the primary regional backend |
| [`adrs/ADR-003-ci-platform.md`](./adrs/ADR-003-ci-platform.md) | GitHub Actions + OIDC as the CI/CD platform |

## How to use this guide

1. **Read [`docs/02-scoring-axes.md`](./docs/02-scoring-axes.md) first.** Decide whether the weighting matches your situation. If you'd weight differently (e.g. you care more about absolute cost than tooling, or you've never used Azure), the recommendation may shift.
2. **Read [`docs/01-recommendation.md`](./docs/01-recommendation.md)** for the chosen stack, *including* the operator-specific caveat at the top.
3. **Read [`docs/03-alternatives-considered.md`](./docs/03-alternatives-considered.md)** to understand what you're giving up. v1.2 expanded this with GitLab, Entra External ID, and Pulumi trade studies.
4. **Use [`docs/decision-trees.md`](./docs/decision-trees.md)** when the doctrine returns "it depends".
5. **Use [`docs/05-phase-plan.md`](./docs/05-phase-plan.md)** as the sequencing playbook — note v1.2 replaces the "1 weekend" framing with honest 1-weekend / 40-80h / 2-4-week ranges.
6. **Before going to production**, read [`07`](./docs/07-origin-security-and-private-networking.md) → [`08`](./docs/08-disaster-recovery-and-backups.md) → [`09`](./docs/09-concentration-risk.md) → [`10`](./docs/10-compliance-and-jurisdiction.md) → [`11`](./docs/11-cicd-and-supply-chain.md). Skipping these is what turns a working prototype into a 3 AM incident.

## Status

| Doc | Status |
|---|---|
| 01 Recommendation | ✅ v1.2 (Hyperdrive vs private-Postgres, Service Bus tier transition, Redis Basic/Standard split, stealth costs) |
| 02 Scoring axes | ✅ v1.2 (Cloudflare deprecation, observability scoring, AI scoring + skills-decay caveat) |
| 03 Alternatives considered | ✅ v1.2 (added GitLab / Entra External ID / Pulumi trade studies) |
| 04 Portability promise | ✅ v1.1 (migration estimates re-baselined) |
| 05 Phase plan | ✅ v1.2 (honest effort ranges, DNSSEC + secondary DNS + hardware 2FA + burn-rate + kill-switch + data-classification) |
| 06 Anti-patterns | ✅ v1.1 (Cosmos APIs, Service Fabric, AKS, Lambda nuance) |
| 07 Origin security + private networking | ✅ v1.2 (corrected AOP mechanism via clientCertificateMode) |
| 08 DR + backups | ✅ v1.2 (corrected Postgres failover CLI, Cloudflare LB URL scope, ACA revision restart syntax, RTO/RPO tier mapping to infra-guide, operator-identity disclosure) |
| 09 Concentration risk | ✅ v1.2 (security-first vs resilience-first variant; Secondary DNS caveat) |
| 10 Compliance + jurisdiction | ✅ v1.1 |
| 11 CI/CD + supply chain | ✅ v1.2 (SLSA L2 vs L3 disclosure, VEX, self-hosted-runner trust-model, runner alternatives comparison) |
| decision-trees | ✅ v1.2 (new — 10 trees) |
| adrs/ | ✅ v1.2 (new — MADR + Nygard reopen-trigger format; 3 exemplar ADRs) |

See [`CHANGELOG.md`](./CHANGELOG.md) for what changed in v1.2 vs v1.1 (5 parallel architect-grade audit passes found 3 P0 runbook bugs + 7 cross-doc contradictions + silent doctrine violations vs upstream guides; v1.2 fixes all P0 + most P1).

## Reviews

This is opinionated, not gospel. Disagreements are welcome via issues — especially with evidence (a benchmark, a billing screenshot, a post-mortem) that contradicts a claim. Quarterly review cadence to incorporate cloud-pricing shifts and new services.

## License

MIT (text + reasoning are free to copy, adapt, and disagree with).
