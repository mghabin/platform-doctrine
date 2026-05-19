# 11 — CI/CD + supply-chain at scale

> GitHub Actions is the right starting point. **At ~10 engineers and ~5+ services**, the bill becomes the second-largest line item on the platform; the controls become the difference between a clean SLSA report and a regulatory finding.

The recommendation in [`01-recommendation.md`](./01-recommendation.md) names GHA + OIDC as the CI/CD baseline. This chapter is the operational follow-through: when to add self-hosted runners, how to manage secrets at scale, how to ship signed artifacts with provenance, and how to handle the org-level concerns the small-team starter skips.

> **Upstream doctrine** — this chapter *applies*, doesn't redefine:
>
> - CI/CD baseline + reusable workflows + OIDC depth — [`infra-engineering-guide/docs/03-ci-cd.md`](https://github.com/mghabin/infra-engineering-guide/blob/main/docs/03-ci-cd.md)
> - Supply-chain controls (SLSA L3 for prod, SBOM, VEX, cosign) — [`infra-engineering-guide/docs/06-security-supply-chain.md`](https://github.com/mghabin/infra-engineering-guide/blob/main/docs/06-security-supply-chain.md) §3-§9
> - Self-hosted runner trust model (`runner_environment=github-hosted` OIDC claim) — `infra-engineering-guide/docs/03-ci-cd.md` §4.2

## CI cost curve

GHA-hosted runners are $0/month at idle, ~$0.008/min on Linux 2-core, scaling up to $0.064/min on 8-core. Sounds cheap until:

| Stage | CI minutes/month | Hosted-runner cost |
|---|---|---|
| Solo (1 service, occasional commits) | ~1,000 | **$8** |
| First product, daily commits + matrix tests | ~10,000 | **$80** |
| 5 services, 10 engineers, full PR matrices + nightly | ~50,000 | **$400+** |
| 20 services, 30 engineers, real CI matrix | ~250,000 | **$2,000+** |

The cost ladder in [`01-recommendation.md`](./01-recommendation.md) doesn't include this. **Build it into your Stage 3+ budget**.

### When to switch off GHA-hosted: compare the alternatives

**Trigger**: monthly GHA-hosted-runner bill exceeds ~$200/month sustained. **Before defaulting to Hetzner self-hosted, compare:**

| Option | Cost (vs hosted baseline) | Ops burden | OIDC trust model | Best for |
|---|---|---|---|---|
| **GHA Larger Runners** (4x / 8x / 16x core, beta-GA 2025) | 1.2-1.8× per-min but eliminate runner ops; usable from any repo without infra | Zero (managed) | Same as standard hosted — preserves `runner_environment=github-hosted` OIDC claim | Modest CI inflation (compute-bound builds), small/medium org, no platform-team |
| **Azure-hosted ephemeral runners** (ACA Jobs running `actions/runner` ephemeral) | ~30-50% cheaper than hosted at sustained load; uses ACA scale-to-zero | Low (you own the image; ACA manages lifecycle) | Self-hosted — does NOT satisfy `runner_environment=github-hosted` claim | Same-cloud locality (faster ACR pulls); want runners to live alongside the rest of Azure |
| **AWS CodeBuild + GHA bridge** | Pay-per-second AWS Compute; competitive at scale | Medium | Self-hosted | Already on AWS-heavy stack |
| **Hetzner CCX self-hosted (this doctrine's pick for large scale)** | **10-100× cheaper** than hosted at >250k min/mo | High (you own patching + isolation + autoscaling) | Self-hosted — **does NOT satisfy `runner_environment=github-hosted`** claim; trust-model implications below | Massive CI workloads (>250k min/mo); have dedicated platform engineer |

**Honest re-evaluation order**: GHA Larger Runners first → Azure-hosted ephemeral → AWS CodeBuild → Hetzner only if the cost gap is materially > $1k/mo *after* trying GHA Larger Runners. Hetzner introduces a 4th vendor (Cloudflare + Azure + GitHub + Hetzner) — the cost win has to be real.

### Self-hosted runner trust-model implications (mandatory disclosure)

The infra-engineering-guide ch03 §4.2 mandates that production deploy workflows MUST require the OIDC claim `runner_environment=github-hosted` to defeat self-hosted-runner abuse (an attacker on a self-hosted runner can otherwise inject their own OIDC claims). If you go self-hosted (Hetzner, Azure-hosted, AWS), you MUST:

1. **Segregate workflows by runner type**: prod-deploy workflows continue to use GHA-hosted runners (preserve the claim); CI/build workflows can use self-hosted.
2. **Add mitigating controls**: every self-hosted runner uses ephemeral mode (`actions/actions-runner` `--ephemeral`), runs in isolated VMs (one job per VM), has read-only access to repos, never holds long-lived secrets.
3. **Network-isolate** self-hosted runners from your Azure prod network (NAT egress only; no inbound; no Private Endpoint access).
4. **Document the trust boundary** in your ADR.

If you can't satisfy these, **don't go self-hosted**. The cost savings aren't worth a Solarwinds-class supply-chain attack vector.

### Recommended self-hosted pattern (if you choose it)

Hetzner CCX (dedicated CPU) bare-metal-ish VMs running `actions/actions-runner` in `--ephemeral` mode in Docker, with [actions-runner-controller](https://github.com/actions/actions-runner-controller) on a small Kubernetes cluster (yes — Kubernetes for *CI* even though the prod recommendation is ACA; the deliberate exception is documented in [`06-anti-patterns.md`](./06-anti-patterns.md) § Silent decisions). Alternative: [philips-labs/terraform-aws-github-runner](https://github.com/philips-labs/terraform-aws-github-runner) for ephemeral AWS-hosted runners. Cost: ~$30-60/month per equivalent-2-core always-on runner.

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

| Control | Tool | SLSA level |
|---|---|---|
| Action pinning to commit SHA | [pinact](https://github.com/suzuki-shunsuke/pinact) CI check | — |
| Container vuln scan | Trivy with `severity: HIGH,CRITICAL` fail-on | — |
| SBOM generation | Syft → upload as build artifact + attach via cosign | — |
| VEX statements (Vulnerability Exploitability eXchange) | OpenVEX via `openvex/vexctl` — generate per-image, attach to the SBOM | Required by `infra-engineering-guide` ch06 §7 |
| Container signing | cosign keyless (Fulcio) + Rekor transparency log | — |
| SLSA provenance (build) | **For new platforms aiming at SLSA Build L3 (infra-guide ch06 §9 "must" for prod)**: use [`slsa-framework/slsa-github-generator`](https://github.com/slsa-framework/slsa-github-generator) — runs builds in an isolated GHA reusable workflow that produces L3-compliant provenance. **For platforms that accept L2** (faster setup, hosted-builder lineage acceptable): `actions/attest-build-provenance`. **Document the choice in your ADR; default to L3 for production artifacts**. | L2 (`attest-build-provenance`) vs L3 (`slsa-github-generator`) |
| Secret scanning | gitleaks pre-commit + CI | — |
| Dependency review | `actions/dependency-review-action` on PRs | — |
| CodeQL | Multi-language analysis | — |
| OSSF Scorecard | Weekly scheduled run | — |
| License compliance | `licensecheck` per language ecosystem | — |
| Markdown/lint | markdownlint-cli2 | — |
| Link check | lychee | — |

All actions pinned to commit SHA. `pinact` enforces this — the platform template's `pinact.yml` workflow fails any PR that introduces a tag-pinned action.

> **`actions/attest-build-provenance` status note (2026)**: GitHub's `attest-build-provenance` reusable action is being superseded by the SLSA framework's native generator + `actions/attest-build-provenance` migration helpers; new platforms should plan to migrate. The L3 path via `slsa-github-generator` is the longer-lived choice.

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
