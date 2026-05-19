# Architectural Decision Records

> One file per load-bearing decision. Format: MADR + Nygard reopen trigger.

This directory complements the narrative chapters in [`../docs/`](../docs). The chapters explain *why* in prose; ADRs make each decision *auditable*: status, date, alternatives, consequences, reopen trigger.

When a decision is corrected in a new doctrine version, the old ADR is marked `Superseded by ADR-NNN` and the new ADR cross-links back. This makes "what changed from v1.0 → v1.1 → v1.2" enumerable instead of buried in prose.

## Template

```markdown
# ADR-NNN — [Short title in imperative form]

**Status:** Accepted | Proposed | Superseded by ADR-NNN | Deprecated
**Date:** YYYY-MM-DD
**Last reviewed:** YYYY-MM-DD
**Supersedes:** ADR-NNN (if applicable)
**Narrative cross-reference:** `docs/NN-section.md`

## Context

What forces / constraints / timing made this decision necessary?
(1–3 sentences)

## Decision

The one-sentence decision.

## Rationale

Why this specific pick over the alternatives.
(2–4 bullets, each citing the scoring axis from `docs/02-scoring-axes.md` if applicable)

## Alternatives considered

| Option | Pro | Con | Why rejected |
|---|---|---|---|
| ... | ... | ... | ... |

## Consequences

| Positive | Negative |
|---|---|
| ... | ... |

## Reopen trigger

Specific, measurable condition under which this ADR should be re-evaluated.
e.g. "Re-open if monthly hosted-runner bill exceeds $1k sustained for 3 months."
```

## Index

| # | Status | Decision | Doc cross-ref |
|---|---|---|---|
| [001](./ADR-001-compute-platform.md) | Accepted | Azure Container Apps as the default compute platform (over AKS, ECS Fargate, Cloud Run) | `docs/01`, `docs/06` |
| [002](./ADR-002-primary-cloud.md) | Accepted | Microsoft Azure as the primary regional backend (over AWS, GCP) | `docs/01`, `docs/02`, `docs/03` |
| [003](./ADR-003-ci-platform.md) | Accepted | GitHub Actions + OIDC as the CI/CD platform (over GitLab CI, Azure DevOps Pipelines, Jenkins) | `docs/11`, `docs/03` |

## How to add a new ADR

1. Pick the next ADR number.
2. Copy the template.
3. Fill it in — be ruthlessly honest about consequences and reopen triggers.
4. Add to the index above.
5. Cross-reference from the relevant narrative chapter (in `docs/NN-…md`).
6. PR + review.

## When to write an ADR

A decision is "load-bearing" (deserves an ADR) when **changing it would meaningfully change the architecture or operational story**. Examples:

- Cloud picker
- Compute / state / queue / cache primitives
- Identity provider
- IaC tool
- CI/CD platform
- Observability backend
- Multi-region trigger
- Major version bumps that change behaviour (not point updates)

A decision is **not** an ADR if it's a routine implementation choice with a clear default (e.g. "use UTF-8", "code formatter is `dotnet format`"). Document those in conventions, not ADRs.

## Standards followed

- [Nygard 2011 — Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
- [MADR (Markdown ADR)](https://adr.github.io/madr/)
- This directory adds explicit **reopen trigger** (per Nygard) which is the single most-missed ADR field in practice.
