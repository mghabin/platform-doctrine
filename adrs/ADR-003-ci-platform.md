# ADR-003 — Use GitHub Actions + OIDC as the CI/CD platform

**Status:** Accepted
**Date:** 2026-05-19
**Last reviewed:** 2026-05-19
**Narrative cross-reference:** [`docs/11-cicd-and-supply-chain.md`](../docs/11-cicd-and-supply-chain.md), [`docs/03-alternatives-considered.md`](../docs/03-alternatives-considered.md) § GitLab

## Context

The doctrine needs a CI/CD platform that supports OIDC federation into Azure (and any future second cloud), preserves the `runner_environment=github-hosted` claim for supply-chain trust, integrates with the supply-chain tooling (`pinact`, `cosign`, Trivy, SBOM, SLSA), and stays available as the team grows from 1 → 50+ engineers.

## Decision

Use **GitHub Actions** as the CI/CD platform, with **OIDC federation** to Azure (Workload Identity Federation → Federated Identity Credential), reusable workflows for org-wide controls, and SHA-pinned third-party actions enforced via `pinact`.

## Rationale

- **OIDC depth into Azure is mature** — `azure/login` action supports the FIC pattern out of the box; same pattern transfers to AWS / GCP / Vault if multi-cloud arrives later.
- **Most code collaboration today happens on GitHub** — switching cost for new hires is real.
- **Reusable workflows + the marketplace** let the platform-template repo (Phase 2 of [`docs/05-phase-plan.md`](../docs/05-phase-plan.md)) enforce org-wide controls without per-product re-implementation.
- **The supply-chain tooling ecosystem is GitHub-first** — `pinact`, `cosign` keyless via Fulcio, GitHub Attestation, OSSF Scorecard, Dependabot grouping — all light up immediately on GHA, with first-class OIDC claims for SLSA provenance.

## Alternatives considered

| Option | Pro | Con | Why rejected (for this stack) |
|---|---|---|---|
| **GitLab CI + GitLab platform** | Operationally integrated (repo + CI + issues + registry + deploy + audit in one product); self-hostable; mature audit/RBAC | Switching cost from GitHub-based collaboration; ecosystem of supply-chain actions is GHA-first | Default to GHA; GitLab is a legitimate alternative for orgs with operational-integration / self-hostable-control-plane / regulatory constraints. See [`docs/03-alternatives-considered.md`](../docs/03-alternatives-considered.md) § GitLab |
| **Azure DevOps Pipelines** | Tight Azure integration; mature enterprise features | Microsoft is investing more in GHA + Copilot than ADO; smaller marketplace; weaker OIDC ecosystem outside Azure | Use only if already-on-ADO |
| **Jenkins** | Self-hosted; complete control | Operational tax; per-team plugin sprawl; not OIDC-native | Not the doctrine's posture |
| **CircleCI / TravisCI / AppVeyor** | Historical alternatives | Smaller marketplace; weaker OIDC depth than GHA | Not competitive in 2026 |

## Consequences

| Positive | Negative |
|---|---|
| OIDC → Azure FIC eliminates long-lived secrets | GitHub's strategic direction is Copilot-driven; pricing for Larger Runners / GHAS / Enterprise may change |
| Reusable workflows let the platform template enforce org-wide controls (SHA pinning, SBOM, cosign, SLSA) | Marketplace dependency means SHA pinning is mandatory (this doctrine enforces via `pinact`) |
| Free for OSS, generous free private-repo minutes | At ~250k CI min/month, GHA-hosted cost crosses ~$2k/mo; self-hosted runners trigger ops-tax discussion (see ADR backlog for runner ADR) |
| First-class OIDC + GitHub Attestation for SLSA provenance | SLSA L3 requires `slsa-framework/slsa-github-generator` (not just `actions/attest-build-provenance`) — see [`docs/11`](../docs/11-cicd-and-supply-chain.md) |

## Reopen trigger

Re-evaluate this ADR if **any of**:

1. **GHA pricing event** — GitHub raises action minutes pricing >2× expected, OR mandates Larger Runners for OIDC OR ties OIDC to GHAS (currently free).
2. **GitHub strategic shift** — GitHub deprecates a feature this doctrine depends on (OIDC, reusable workflows, GitHub Attestation), OR forces a bundled product (e.g. Copilot or GHAS required for OIDC).
3. **Org operational-integration pressure** — team grows past 50 engineers AND the platform team explicitly requests GitLab for audit/integration reasons.
4. **Regulatory** — a compliance requirement forces a self-hostable CI control-plane (uncommon outside defence/financial).
5. **Cost crossover** — sustained CI cost (hosted runners + GHAS if used) crosses 1.5× GitLab Premium's equivalent for the same team size. Don't switch on a single-month spike — sustain 3 months.

When re-evaluating, the natural migration target is **GitLab CI** with self-hosted runners on the same Hetzner pattern documented in [`docs/11`](../docs/11-cicd-and-supply-chain.md). GitLab-CI's `id_tokens` keyword provides equivalent OIDC depth to GHA, so the federation pattern transfers.
