# ADR-001 — Use Azure Container Apps as the default compute platform

**Status:** Accepted
**Date:** 2026-05-19
**Last reviewed:** 2026-05-19
**Narrative cross-reference:** [`docs/01-recommendation.md`](../docs/01-recommendation.md) (picks table — Backend APIs / web apps), [`docs/06-anti-patterns.md`](../docs/06-anti-patterns.md) § `AKS from day one`

## Context

The platform doctrine recommends a stack designed to be cold-cheap (`$0` idle while building) and to scale to multi-product org without rewriting. The choice of compute primitive is load-bearing: it dictates the deployment model, autoscaling behaviour, networking story, observability integration, and the IaC modules that every product inherits.

## Decision

Use **Azure Container Apps (Consumption plan)** as the default compute primitive for every web app, API, and worker. ACA Jobs (`triggerType: Schedule` or KEDA-driven) for crons and queue-consumers.

## Rationale

- **Scale-to-zero is real** (`min=0` per service) — matches the `$0 idle` cost ceiling that's a stated goal (axis 1 stability + axis 10 pricing predictability).
- **Multi-revision + traffic-splitting + MI/FIC** are first-class — covers the EntraAuthPatterns auth doctrine without bolt-on (axis 12 existing skills).
- **Lower operational tax than AKS** for the workloads the recommendation actually targets (axis 9 DX).
- **Container images are portable** to Fargate / Cloud Run / Fly.io / k8s anywhere — preserves the portability promise in [`docs/04-portability-promise.md`](../docs/04-portability-promise.md).

## Alternatives considered

| Option | Pro | Con | Why rejected |
|---|---|---|---|
| **AKS day-1** | Industry-standard primitives; future-proof for arbitrary scale | Operational tax (cluster upgrades, node-pool sizing, networking, CSI) is real before any app code; steeper learning curve | Premature for 1-product / solo-to-small-team; container images are portable so AKS migration later is bounded |
| **Azure Functions (Consumption)** | $0 idle; quick hello-world | Binding lock-in, cold-start variability, runtime model couples to Azure | Bindings are the lock-in; not portable; use ACA Jobs for the same use case |
| **Azure App Service Plans** | Mature; lots of tutorials | Pay-per-instance even idle; no scale-to-zero; no multi-revision traffic split | Strictly worse than ACA for new workloads |
| **AWS ECS Fargate** | Most mature container PaaS; deep AWS ecosystem | Adds AWS to a stack otherwise on Azure; existing-skills tax | Off-stack — only relevant if we'd chosen AWS as primary cloud (see ADR-002) |
| **GCP Cloud Run** | Best DX of any container PaaS; per-request billing | Adds GCP to a stack otherwise on Azure; GCP deprecation track-record concern | Off-stack |
| **Self-hosted Kubernetes on Hetzner** | Cheapest raw $; full control | Ops tax for solo-to-org builders; not the doctrine's posture | Out of scope; CI runners are the only exception |

## Consequences

| Positive | Negative |
|---|---|
| `$0` idle for non-traffic apps via `min=0` | CPU/memory-based scaling rules **cannot** scale to 0 — only HTTP/event-driven do |
| Multi-revision traffic splitting for blue/green deploys | KEDA scaler limits at very high event-throughput (rare; manageable) |
| MI/FIC integration "just works" with the EntraAuthPatterns auth library | If you need sidecars / CRDs / multi-namespace tenancy / advanced networking, you eventually hit a ceiling |
| Container images stay portable | ACA-specific Bicep properties create some lock-in (manageable; covered by ADR-006 IaC choice) |

## Reopen trigger

Re-evaluate this ADR if **any of**:

1. ACA imposes a hard ceiling that affects a specific live workload (specific examples to watch: 1000-rps-per-app cap with current pricing; sidecar limitations beyond what ACA's GA-sidecar-support covers; advanced VNet topology we can't express).
2. AKS Autopilot-equivalent reaches feature-parity with ACA on scale-to-zero + per-revision routing + MI access.
3. Team grows past **20 services** AND we genuinely need a platform-team's worth of Kubernetes capabilities (CRDs, multi-tenancy via namespaces, service mesh, GPU scheduling).
4. Microsoft signals reduced investment in ACA (e.g. >12 months without meaningful feature shipping).

If we trigger #1 or #3, the migration target is **AKS** (Microsoft path-of-least-resistance) — see [`mghabin/dotnet-engineering-guide` chapter 06](https://github.com/mghabin/dotnet-engineering-guide/blob/main/docs/06-cloud-native.md) for the .NET-on-AKS playbook. Container images are unchanged; the IaC is the migration cost.
