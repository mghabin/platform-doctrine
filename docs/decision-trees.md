# Decision trees

> When the doctrine says "it depends", these are the trees that the dependency hinges on. Each tree returns a specific pick + the ADR or chapter that owns the full rationale.

## 1. Cloud picker

```text
Q1. Do you have meaningful existing skill on a specific cloud (>40h direct work)?
├── YES → that cloud is the default for year-1 (avoid relearning tax)
│         see ADR-002; re-score annually as the team hires
└── NO → continue
   Q2. What's your dominant priority?
   ├── Stability + biggest talent pool → AWS (Cloudflare + AWS)
   ├── Best DX + best containers + accept GCP deprecation risk → GCP (Cloudflare + GCP)
   └── Strongest enterprise identity bundle + AI governance → Azure (Cloudflare + Azure)
   Then: re-run scoring in `docs/02-scoring-axes.md` with your weights.
```

Owners: [`docs/01-recommendation.md`](./01-recommendation.md), [`docs/02-scoring-axes.md`](./02-scoring-axes.md), [`docs/03-alternatives-considered.md`](./03-alternatives-considered.md), [ADR-002](../adrs/ADR-002-primary-cloud.md).

---

## 2. Compute platform picker

```text
Q1. Greenfield or existing?
├── EXISTING on AKS/GKE/EKS — stay; doctrine doesn't force you off Kubernetes
└── GREENFIELD → continue
   Q2. Do you NEED any of: custom CRDs, multi-namespace tenancy, GPU
       scheduling, sidecars beyond platform-default, advanced VNet that ACA
       doesn't expose?
   ├── YES → AKS (managed Kubernetes); see dotnet-engineering-guide ch06
   └── NO → ACA (default); see ADR-001
   Q3. Workload pattern?
   ├── Cron / batch / probe-and-exit → ACA Jobs
   ├── HTTP API / web app → ACA service
   ├── Long-running worker (queue consumer) → ACA service with KEDA scaler on Service Bus
   └── Edge-cacheable / sub-50ms global → Cloudflare Workers
```

Owners: [`docs/01-recommendation.md`](./01-recommendation.md), [`docs/06-anti-patterns.md`](./06-anti-patterns.md), [ADR-001](../adrs/ADR-001-compute-platform.md).

---

## 3. State picker

```text
Q1. Is your workload globally distributed multi-write with measured low-latency
    requirements that Postgres + read replicas can't meet?
├── YES → Cosmos DB serverless (accept proprietary-API lock-in; document in ADR)
└── NO → continue
   Q2. Relational + transactions + JOINs + measured complex queries?
   ├── YES → Postgres Flexible Server (Burstable B1ms non-prod, GP for prod)
   └── NO → continue
   Q3. Document / key-value with known access patterns + need sub-ms reads at scale?
   ├── YES → measure first; default Postgres JSONB; if measurement shows insufficient,
   │         use Cosmos DB (Azure) or DynamoDB On-Demand (if multi-cloud)
   └── NO → Postgres + JSONB (default for everything that isn't decidedly distributed)
   Q4. Object / blob / file?
   ├── Customer-facing assets → Cloudflare R2 (zero egress)
   └── Backups + Azure-internal → Azure Blob Cool/Archive
```

Owners: [`docs/01-recommendation.md`](./01-recommendation.md), [`docs/06-anti-patterns.md`](./06-anti-patterns.md) § Cosmos DB / DynamoDB.

---

## 4. Identity provider picker (customer-facing)

```text
Q1. Sovereignty / sanctions concern + need to decouple from your cloud-provider's
    compliance obligations?
├── YES → Keycloak self-hosted (EU sovereignty) OR FusionAuth
└── NO → continue
   Q2. Prototype phase or pre-revenue (≤10k MAU expected this year)?
   ├── YES → Auth0 (fastest to ship; OIDC-standard; cheapest portability cost)
   └── NO → continue
   Q3. Microsoft-stack purity preference + one-throat-to-choke + no sovereignty?
   ├── YES → Entra External ID (legitimate but accept blast-radius coupling — record in ADR)
   └── NO → Keycloak (cost-stable; have ops capacity) OR FusionAuth (self-hosted DX without Keycloak depth)
```

Owners: [`docs/01-recommendation.md`](./01-recommendation.md), [`docs/03-alternatives-considered.md`](./03-alternatives-considered.md) § Entra External ID.

---

## 5. CI runner picker (when to leave GHA-hosted)

```text
Q1. Monthly GHA-hosted bill < $200/month sustained?
├── YES → stay on GHA-hosted; don't optimize prematurely
└── NO → continue
   Q2. Bill < $1000/month?
   ├── YES → GHA Larger Runners (4x/8x/16x) — eliminate runner ops, modest cost increase
   └── NO → continue
   Q3. Already on Azure-heavy stack + want runners close to ACR?
   ├── YES → Azure-hosted ephemeral runners via ACA Jobs running actions/runner --ephemeral
   └── NO → continue
   Q4. Already on AWS or willing to add it?
   ├── YES → AWS CodeBuild + GHA bridge OR philips-labs/terraform-aws-github-runner
   └── NO → Hetzner self-hosted (accept 4th vendor + trust-model implications: see docs/11)
   Q5. Did you preserve `runner_environment=github-hosted` claim for prod deploys?
   ├── YES → proceed
   └── NO → STOP — you have a supply-chain attack vector; see docs/11 §self-hosted-runner-trust-model
```

Owners: [`docs/11-cicd-and-supply-chain.md`](./11-cicd-and-supply-chain.md).

---

## 6. DR strategy picker

```text
Q1. What's the highest RTO / RPO your SLA contract allows?
├── < 30 min RTO + < 5 min RPO → Tier 1: cross-region active-passive with Postgres async replica
├── < 4h RTO + < 1h RPO → Tier 2: cross-region snapshot
└── < 24h RTO + < 24h RPO → Tier 3: single-region daily backup
   Then: drill the chosen tier quarterly; document in docs/08.
```

Owners: [`docs/08-disaster-recovery-and-backups.md`](./08-disaster-recovery-and-backups.md), [`infra-engineering-guide/docs/08-data-state.md`](https://github.com/mghabin/infra-engineering-guide/blob/main/docs/08-data-state.md) §4.

---

## 7. Edge layer picker

```text
Q1. Do you have a hard requirement for Azure Private Link to an internal-only Azure service?
├── YES → Azure Front Door (it can integrate via Private Link; Cloudflare can't)
└── NO → continue
   Q2. Optimize for cost + global reach + R2 zero-egress?
   ├── YES → Cloudflare (default for this doctrine)
   └── NO (you want same-cloud edge with same support contract / audit trail)
      → Azure Front Door OR (if AWS) CloudFront — accept higher cost + smaller talent pool
```

Owners: [`docs/01-recommendation.md`](./01-recommendation.md), [`docs/06-anti-patterns.md`](./06-anti-patterns.md) § Front Door.

---

## 8. Multi-region trigger picker

```text
Q1. Do you have prod customers in more than one geography?
├── NO → single-region (Phase 3 of docs/05); don't pre-pay multi-region cost
└── YES → continue
   Q2. Is your SLA contract enforcing cross-region availability?
   ├── YES → Phase 4: Cloudflare LB + Postgres cross-region replica + ACA env in second region
   └── NO → Phase 4 only if your incident-history shows regional outages affecting customers
```

Owners: [`docs/05-phase-plan.md`](./05-phase-plan.md), [`docs/08-disaster-recovery-and-backups.md`](./08-disaster-recovery-and-backups.md).

---

## 9. IaC tool picker

```text
Q1. Single-cloud (Azure only)?
├── YES → Bicep + Azure Verified Modules + CAF AVM accelerator
└── NO → continue
   Q2. Multi-cloud and team prefers HCL?
   ├── YES → OpenTofu (Linux Foundation; broad provider support)
   └── NO (team has strong .NET / TS / Python culture and accepts Pulumi Cloud as state vendor)
      → Pulumi
```

Owners: [`docs/01-recommendation.md`](./01-recommendation.md), [`docs/03-alternatives-considered.md`](./03-alternatives-considered.md) § Pulumi.

---

## 10. SLSA level picker

```text
Q1. Production artifacts that ship to customers OR a regulatory regime requiring provenance?
├── YES → SLSA Build L3 (slsa-framework/slsa-github-generator)
│         see docs/11 supply-chain controls
└── NO → SLSA Build L2 (actions/attest-build-provenance)
         document the L2 exception + upgrade path in ADR
```

Owners: [`docs/11-cicd-and-supply-chain.md`](./11-cicd-and-supply-chain.md), [`infra-engineering-guide/docs/06-security-supply-chain.md`](https://github.com/mghabin/infra-engineering-guide/blob/main/docs/06-security-supply-chain.md) §9.

---

## How to read these trees

Every leaf returns a specific pick. Every pick is owned by a narrative chapter and (for load-bearing decisions) an ADR. If a tree returns an answer you disagree with, that's the start of a doctrine conversation — open a PR re-scoring the relevant axes in [`02-scoring-axes.md`](./02-scoring-axes.md), not a one-off override.
