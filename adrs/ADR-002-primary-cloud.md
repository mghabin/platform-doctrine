# ADR-002 — Use Microsoft Azure as the primary regional backend

**Status:** Accepted (operator-specific)
**Date:** 2026-05-19
**Last reviewed:** 2026-05-19
**Narrative cross-reference:** [`docs/01-recommendation.md`](../docs/01-recommendation.md), [`docs/02-scoring-axes.md`](../docs/02-scoring-axes.md), [`docs/03-alternatives-considered.md`](../docs/03-alternatives-considered.md)

## Context

The doctrine needs one primary regional cloud for stateful workloads (Postgres, queues, secrets, observability, AI). The choice shapes IaC, hiring, support contracts, compliance posture, and the operational story for the next 5-10 years.

## Decision

Use **Microsoft Azure** as the primary regional backend. **Cloudflare** fronts the edge (see separate ADR — not yet written, lives implicitly in [`docs/01`](../docs/01-recommendation.md) and [`docs/09`](../docs/09-concentration-risk.md)).

## Rationale

- **Best cloud-native telemetry stack** for the workloads this doctrine targets — App Insights + Log Analytics + KQL is the strongest *included* observability bundle (axis 5 telemetry, axis 6 monitoring + alerts).
- **Best AI access for Microsoft-stack shops** — Azure OpenAI's VNet/Private Link/CMK/PTU + Entra RBAC is unmatched for governed AI workloads (axis 7 AI scaling).
- **Tooling DX** — Bicep + `az` CLI + VS Code + devcontainers + GitHub-native ergonomics is the strongest cloud-native tooling bundle (axis 4 tooling).
- **Existing operator skills** — the founder has 100+ hours of Azure muscle memory (axis 12). **This factor is asymmetric and decays as the team hires** — see reopen trigger.

## Alternatives considered

| Option | Pro | Con | Why rejected |
|---|---|---|---|
| **AWS** | Best stability record (axis 1), largest talent pool (axis 11), most mature container PaaS (Fargate) | Worst DX of big 3 (axis 9); CDK/CloudFormation/IAM are real productivity cost; no existing-skills advantage | Skills tax + DX tax for an operator already fluent in Azure; the stability gap is dwarfed by 6-12 months of "AWS-that-you're-learning" misconfiguration |
| **GCP** | Best DX (axis 9); Cloud Run is best-in-class container PaaS; cleanest IAM | GCP product-deprecation track record (Cloud IoT Core 2023, App Engine cycles) is a real signal at 10-year horizon; smaller talent pool | Deprecation risk + smaller talent pool + no existing-skills advantage |
| **Hetzner / OVH self-hosted** | Cheapest raw $; EU sovereignty for some shops | Ops tax (patching, networking, HA, region setup); not the "never regret" path for solo-to-org builders | Out of scope for the operator profile |

## Consequences

| Positive | Negative |
|---|---|
| Operator ships features Day 1 (no relearning curve) | Azure had bigger headline outages than AWS in 2024 (mitigated by NOT using Front Door, Cosmos, Functions Consumption — the worst-track-record pieces) |
| App Insights + LAW + KQL covers observability without a 3rd-party APM bill | KQL queries + Workbooks are Azure-specific (medium portability lock-in) |
| EntraAuthPatterns code + Bicep modules transfer 1:1 | If the team grows beyond ~5 engineers and none come pre-trained on Azure, the existing-skills advantage erodes |
| Azure OpenAI for governed AI / data residency / Private Link | Microsoft strategic direction (Copilot-first, OpenAI partnership volatility) carries some uncertainty for 10-year horizon |

## Reopen trigger

Re-evaluate this ADR if **any of**:

1. **Team composition changes** — the org hires >5 engineers without prior Azure experience AND they're spending >20% of their time fighting Azure-specific quirks. Hiring market is Azure-thin in your geography compared to AWS.
2. **Material Azure outage** — Azure has a >24h cross-region incident in a 12-month window that AWS demonstrably wouldn't have had (e.g. an Entra-cascade that takes down something the doctrine said wouldn't be affected).
3. **Microsoft strategic shift** — Microsoft signals reduced infrastructure investment (e.g. deprioritizes ACA / Postgres Flex in favour of Copilot-first products); 12-month signal window.
4. **Pricing event** — Azure pricing changes that materially break the cost ladder in [`docs/01-recommendation.md`](../docs/01-recommendation.md) (e.g. ACA per-request pricing jumps >50%, Postgres Flex Burstable B1ms doubles).
5. **A specific workload** the org must build cannot be done well on Azure (specific enough to be measurable, not "I prefer AWS").

**When re-evaluating, run the scoring exercise in [`docs/02-scoring-axes.md`](../docs/02-scoring-axes.md) again with the team's *current* axis weights (not the founder's original weights).** The right answer in year 3 may differ from year 0.
