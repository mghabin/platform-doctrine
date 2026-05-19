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

| Doc | Purpose |
|---|---|
| [`docs/01-recommendation.md`](./docs/01-recommendation.md) | The full opinionated recommendation with architecture, picks per capability, cost ladder, phase plan. |
| [`docs/02-scoring-axes.md`](./docs/02-scoring-axes.md) | The criteria and weights used to make the call. Re-runnable for different operator profiles. |
| [`docs/03-alternatives-considered.md`](./docs/03-alternatives-considered.md) | Honest write-ups of the candidates that lost (AWS, GCP, all-Cloudflare, Hetzner, Fly.io). Why each lost on the criteria above. |
| [`docs/04-portability-promise.md`](./docs/04-portability-promise.md) | The explicit "if [vendor] turns hostile" test applied to every choice. What's safe; what's lock-in; what's the cost to leave. |
| [`docs/05-phase-plan.md`](./docs/05-phase-plan.md) | Concrete sequencing from solo-builder ($0 idle) to multi-region org. |
| [`docs/06-anti-patterns.md`](./docs/06-anti-patterns.md) | Services explicitly **avoided** in the recommended stack, and why. Avoiding the wrong thing is half the value. |

## How to use this guide

1. **Read [`docs/02-scoring-axes.md`](./docs/02-scoring-axes.md) first.** Decide whether the weighting matches your situation. If you'd weight differently (e.g. you care more about absolute cost than tooling), the recommendation may shift.
2. **Read [`docs/01-recommendation.md`](./docs/01-recommendation.md)** for the chosen stack.
3. **Read [`docs/03-alternatives-considered.md`](./docs/03-alternatives-considered.md)** to understand what you're giving up.
4. **Use [`docs/05-phase-plan.md`](./docs/05-phase-plan.md)** as the sequencing playbook.

## Status

| Doc | Status |
|---|---|
| 01 Recommendation | ✅ Shipped |
| 02 Scoring axes | ✅ Shipped |
| 03 Alternatives considered | ✅ Shipped |
| 04 Portability promise | ✅ Shipped |
| 05 Phase plan | ✅ Shipped |
| 06 Anti-patterns | ✅ Shipped |

## Reviews

This is opinionated, not gospel. Disagreements are welcome via issues — especially with evidence (a benchmark, a billing screenshot, a post-mortem) that contradicts a claim. Quarterly review cadence to incorporate cloud-pricing shifts and new services.

## License

MIT (text + reasoning are free to copy, adapt, and disagree with).
