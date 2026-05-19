# 11 — CI/CD + supply-chain at scale

> GitHub Actions is the right starting point. **At ~10 engineers and ~5+ services**, the bill becomes the second-largest line item on the platform; the controls become the difference between a clean SLSA report and a regulatory finding.

The recommendation in [`01-recommendation.md`](./01-recommendation.md) names GHA + OIDC as the CI/CD baseline. This chapter is the operational follow-through: when to add self-hosted runners, how to manage secrets at scale, how to ship signed artifacts with provenance, and how to handle the org-level concerns the small-team starter skips.

## CI cost curve

GHA-hosted runners are $0/month at idle, ~$0.008/min on Linux 2-core, scaling up to $0.064/min on 8-core. Sounds cheap until:

| Stage | CI minutes/month | Hosted-runner cost |
|---|---|---|
| Solo (1 service, occasional commits) | ~1,000 | **$8** |
| First product, daily commits + matrix tests | ~10,000 | **$80** |
| 5 services, 10 engineers, full PR matrices + nightly | ~50,000 | **$400+** |
| 20 services, 30 engineers, real CI matrix | ~250,000 | **$2,000+** |

The cost ladder in [`01-recommendation.md`](./01-recommendation.md) doesn't include this. **Build it into your Stage 3+ budget**.

### When to switch to self-hosted runners

**Trigger**: monthly GHA-hosted-runner bill exceeds $200/month sustained.

**Recommended pattern**: Hetzner CCX (dedicated CPU) bare-metal-ish VMs running `actions/actions-runner` in Docker, with [actions-runner-controller](https://github.com/actions/actions-runner-controller) on Kubernetes or [philips-labs/terraform-aws-github-runner](https://github.com/philips-labs/terraform-aws-github-runner) for ephemeral runners. Cost: ~$30-60/month per always-on runner equivalent to a hosted 2-core minute pool. **10-100x cheaper than hosted** at scale.

**Trade-off**: you now own runner patching, autoscaling, container isolation, ephemeral-runner lifecycle. Worth it past $200/mo; not worth it below.

**Where**: Hetzner Falkenstein/Helsinki/Ashburn for cheap CPU; the runners reach Azure/Cloudflare over the public internet, no special networking needed. They are *not* part of the production platform — keep the operational concerns separate.

### Cache wisely

GHA's built-in cache + `actions/cache` saves 30-60% of CI time on most workloads. Per-language tips:

- **.NET**: cache `~/.nuget/packages` keyed on `**/packages.lock.json` hash
- **Node**: cache `~/.npm` keyed on `**/package-lock.json` hash
- **Docker**: BuildKit cache mounts + GHA cache backend (`type=gha`) — speeds rebuilds 10x

## Secret management at scale

The pattern that worked at 1 service breaks at 20.

### Phase 1-2: GHA secrets + OIDC + Azure Key Vault

- Repo / environment secrets for the few static values (registry credentials if any, sometimes Cloudflare API tokens for IaC).
- Everything else via OIDC → Azure AD → Key Vault. **Never** put production secrets in GHA env secrets.

### Phase 3+: organization-level secret store

Once 5+ repos consume the same secrets, the per-repo `gh secret set` pattern becomes a typo-driven incident factory. Use:

- **GitHub Organization Secrets** with repo-allowlist for the *small* set of truly shared values (Cloudflare API token, package registry tokens).
- **Centralized secret rotation runbook** committed to the platform-template repo. One quarter = one secret-rotation drill.
- **Audit logs** streamed from GH org → Log Analytics for "who created/read secret X?".

### Secret-rotation cadence

| Secret type | Rotation cadence |
|---|---|
| Cloudflare API tokens (CI-scoped) | Quarterly |
| Azure Key Vault keys (CMK if used) | Annually |
| Postgres admin password | Annually (or never, if MI-only access enforced) |
| Federated Identity Credentials (FICs) | No rotation needed — short-lived by design |
| GHA repo secrets | Quarterly if any |
| Cosign signing keys | Annual; or use keyless signing (Fulcio) and skip |

## Supply-chain controls

The dotnet-engineering-guide and infra-engineering-guide already cover most of these. The platform-doctrine perspective: **enforce them at the platform template level, not per-product**.

### Bake into the reusable GHA workflow

The `mghabin/cloudflare-azure-platform` template (Phase 2) ships a reusable workflow that every product inherits. It enforces:

| Control | Tool |
|---|---|
| Action pinning to commit SHA | [pinact](https://github.com/suzuki-shunsuke/pinact) CI check |
| Container vuln scan | Trivy with `severity: HIGH,CRITICAL` fail-on |
| SBOM generation | Syft → upload as build artifact |
| Container signing | cosign keyless (Fulcio) + Rekor transparency log |
| SLSA provenance | `actions/attest-build-provenance` |
| Secret scanning | gitleaks pre-commit + CI |
| Dependency review | `actions/dependency-review-action` on PRs |
| CodeQL | Multi-language analysis |
| OSSF Scorecard | Weekly scheduled run |
| License compliance | `licensecheck` per language ecosystem |
| Markdown/lint | markdownlint-cli2 |
| Link check | lychee |

All actions pinned to commit SHA. `pinact` enforces this — the platform template's `pinact.yml` workflow fails any PR that introduces a tag-pinned action.

### Verify-at-deploy, not just verify-at-build

The deploy workflow (separate from build) verifies:

1. The container image was built by a known build workflow on this repo (`gh attestation verify --predicate-type ...`).
2. The cosign signature matches a known Fulcio identity (your GHA workflow's OIDC identity).
3. The SBOM is attached and contains no known critical CVEs since build time.

If any check fails, the deploy stops — even if the image is already in the registry. This catches "attacker uploaded a malicious image with a stolen ACR push token" between build and deploy.

## Dependency management at scale

| Tool | When |
|---|---|
| Dependabot (built-in) | Solo → small org. Group by ecosystem to avoid PR explosion. |
| Renovate | When Dependabot's grouping isn't expressive enough (e.g., you want to bundle "all OpenTelemetry packages" across multiple lock files). |
| Centralized PR templates | Standard "what changed / how tested / rollback plan" for every dependency PR. |

Dependabot grouping that's worked at scale (from the EntraAuthPatterns repo):

```yaml
groups:
  microsoft-identity:
    patterns: ["Microsoft.Identity.*", "Microsoft.IdentityModel.*"]
  aspnetcore:
    patterns: ["Microsoft.AspNetCore.*"]
  opentelemetry:
    patterns: ["OpenTelemetry.*"]
  azure:
    patterns: ["Azure.*", "Microsoft.Azure.*"]
  analyzers:
    patterns: ["Meziantou.Analyzer", "SonarAnalyzer.*", "StyleCop.*"]
  test:
    patterns: ["xunit*", "Microsoft.NET.Test.Sdk", "Microsoft.Testing.*", "NSubstitute", "Shouldly"]
  actions:
    patterns: ["*"]  # all GitHub Actions
```

This collapses 30+ Dependabot PRs per week into ~6 grouped ones.

## What this stack does NOT use

- **Jenkins / TeamCity / GitLab CI** — fine if you're already on them; not what we'd choose new. GHA's GitHub-native ergonomics + marketplace + OIDC depth are the deciding factor.
- **CircleCI / Travis CI / AppVeyor** — historical, no longer competitive.
- **Azure DevOps Pipelines** — still works, but Microsoft is investing more in GHA. New orgs should start on GHA.
- **Bazel for monorepo builds** — premature for <50 engineers. When you cross that line, consider it.

## Phase gates

| Phase | CI/CD gate |
|---|---|
| Phase 1 | OIDC → Azure FIC, repo secret budget alerts at $50/$200/$500 |
| Phase 2 | Reusable workflow with: pinact, Trivy, cosign keyless, SBOM, SLSA, secret scan |
| Phase 3 | Dependabot grouping rules in place; secret rotation runbook committed |
| Phase 4 | Self-hosted runners on Hetzner once GHA bill > $200/mo sustained |
| Phase 5 | Organization secrets; SOC 2 audit-evidence collection automated |
| Phase 7 | Annual supply-chain attack-simulation drill (e.g. mint a fake provenance, verify deploy rejects it) |

## Sources

- SLSA framework — [slsa.dev](https://slsa.dev/)
- cosign keyless signing — [github.com/sigstore/cosign](https://github.com/sigstore/cosign)
- GitHub Actions OIDC to Azure — [docs.github.com/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure](https://docs.github.com/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure)
- pinact — [github.com/suzuki-shunsuke/pinact](https://github.com/suzuki-shunsuke/pinact)
- actions-runner-controller — [github.com/actions/actions-runner-controller](https://github.com/actions/actions-runner-controller)
- philips-labs/terraform-aws-github-runner — [github.com/philips-labs/terraform-aws-github-runner](https://github.com/philips-labs/terraform-aws-github-runner)
- Hetzner Cloud pricing (CCX dedicated CPU) — [hetzner.com/cloud](https://www.hetzner.com/cloud)
- OSSF Scorecard — [securityscorecards.dev](https://securityscorecards.dev/)
