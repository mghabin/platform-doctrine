# 04 — Portability promise

> The explicit "if [vendor] turns hostile" test, applied to every choice in [`01-recommendation.md`](./01-recommendation.md). What's safe; what's lock-in; what's the cost to leave.

A foundation that can't be left isn't a foundation — it's a hostage. The recommended stack is designed so that every primary choice can be migrated in **weeks, not years**.

## The test

For each component, answer:

1. **If the vendor doubled prices tomorrow, what happens?**
2. **If the vendor was acquired and deprecated this product, what happens?**
3. **If the vendor had a multi-day outage, can we fail over?**
4. **If we had to migrate everything off, how long?**

## Component-by-component

| Component | Vendor lock-in? | If you had to leave, what's the cost? |
|---|---|---|
| **OCI container images** | None | Run on AWS Fargate / GCP Cloud Run / Fly.io / k8s anywhere. **No code change.** |
| **Postgres on Azure Flexible Server** | None | `pg_dump` + restore to any cloud's managed Postgres or self-hosted. **No code change.** |
| **Cloudflare R2 (S3 API)** | Light | Already S3-API; swap backup target to S3 or GCS or MinIO. **Endpoint config change.** |
| **OpenTelemetry SDKs** | None | Re-point `OTEL_EXPORTER_OTLP_ENDPOINT` env var at Grafana Cloud / Datadog / Honeycomb / New Relic. **Config only.** |
| **Bicep IaC** | Azure-only | ~2 week rewrite to OpenTofu. Modules are <300 lines each. The patterns transfer directly. |
| **GitHub Actions OIDC** | None | Re-pointable to any cloud's federated identity in a day. |
| **Azure Key Vault for secrets** | Light | Secrets stored as opaque blobs; export + import to AWS Secrets Manager / GCP Secret Manager / Vault. **A day's work per env.** |
| **Service Bus for queues** | Medium | Message contract is yours; producer/consumer code is library-thin. Swap to SQS / Pub/Sub / RabbitMQ in **~1 week**. |
| **App Insights / Log Analytics / KQL** | Medium (queries are KQL-specific) | Telemetry collection (OTel) is portable; **dashboards and KQL queries need rewrite** to Datadog/Grafana. ~2 weeks of dashboard/alert porting. |
| **Azure Container Apps configuration** | Medium | YAML/Bicep is Azure-shaped but maps cleanly to Cloud Run / Fargate task definitions. **~1 week to port.** |
| **Managed Identity / Federated Credentials** | Medium | Code uses `DefaultAzureCredential` → swap to `WorkloadIdentityFederation` SDK on AWS/GCP. **~3-5 days for app code; longer for CI/CD wiring.** |
| **Azure OpenAI** | Light | Same OpenAI API shape; swap endpoint + key to OpenAI direct / Anthropic / Bedrock. **Hours, not days.** |
| **Cloudflare** ⚠️ | High — and intentional | The edge layer is not portable. **You would never want to leave it.** Cloudflare's positioning is "sit in front of any backend" — they never compete for the backend itself, so the lock-in is the price for the best edge in the industry. |

## What you would genuinely regret if locked-in

The recommended stack **deliberately avoids** the Azure primitives with high lock-in cost:

| Avoided service | Why it would be regrettable |
|---|---|
| **Cosmos DB** | Proprietary API; pricing model hostile; migrating off requires rewriting all data access. |
| **Logic Apps / Power Automate** | Proprietary DSL; not portable. Write workflow code in your app instead. |
| **Azure Functions (Consumption tier) with Functions-specific bindings** | Runtime lock-in; trigger semantics not portable. Use ACA Jobs (containers, your code). |
| **Front Door (with custom rules)** | Locks you into Azure's edge; replicating rules elsewhere is real work. Cloudflare instead. |
| **Service Fabric** | Effectively dead; migration target unclear. Never start here. |
| **API Management (with custom policy XML)** | Proprietary policy language. Use a code-based API gateway instead. |

## Migration cost summary

A full migration from this stack to any other major cloud:

| Component | Effort |
|---|---|
| Container images | 0 (run as-is) |
| Postgres data | ~1 day per DB (dump + restore + verify) |
| Object storage (R2 stays) | 0 (R2 isn't tied to Azure) |
| Secrets export + reimport | ~1 day per env |
| IaC rewrite (Bicep → OpenTofu) | ~2 weeks for the full module set |
| CI/CD re-wiring | ~3-5 days (GHA OIDC change) |
| App code changes (`DefaultAzureCredential` → equivalent) | ~3-5 days |
| Observability dashboards + alerts re-port | ~2 weeks |
| Network + DNS cutover | ~1 day (Cloudflare LB makes this trivial) |
| **Total for a small org** | **~4-6 weeks** |
| **Total for a multi-product org with 5+ services** | **~2-3 months** |

This is dramatically lower than the typical "we're locked in for years" cloud migration. The reason: every primary choice was made on portable primitives (containers, Postgres, S3-API, OTel) rather than proprietary services.

## The Cloudflare lock-in caveat

Cloudflare is the **one intentional lock-in** in the stack. The reasoning:

- The portability test is **"could you move if you needed to?"** — and for Cloudflare, the answer is yes (you can put any CDN in front of any origin), but you'd give up genuine quality + cost advantages.
- Cloudflare's commercial model is to *sit in front of any backend*, not to compete with backends. So they never have an incentive to weaponize the lock-in by deprecating backend access.
- Cloudflare's product expansion pattern is additive — they ship new primitives (Workers AI, Hyperdrive, Pages) without breaking old ones. This is the opposite of GCP's pattern.
- R2's zero-egress fee is the single biggest cost moat in the architecture. The alternative (any other CDN + paying for egress from the backend) is materially more expensive.

This is the kind of lock-in that's **economically rational to embrace** — when leaving costs you more than staying does. As long as that remains true, the lock-in is benign.

## Annual portability dry-run (recommended)

Once per year, pick one workload and deploy it to a different cloud from the same container image. This:

- Validates your portability claims are still true.
- Surfaces hidden Azure-isms you've accumulated (e.g. `DefaultAzureCredential` that secretly requires `AZURE_TENANT_ID`).
- Keeps your migration runbook fresh.
- Costs ~1 weekend of effort per year.

Suggested target: **Fly.io**. Reasons: best DX of any alternative, low setup cost, validates that your container is genuinely portable, and Fly.io's regional model exercises a different load-balancer / Postgres-pooler path than Azure.

## Sources

- 12-factor app principles (the OG portability checklist) — [12factor.net](https://12factor.net/)
- OpenTelemetry as portability layer — [opentelemetry.io](https://opentelemetry.io/)
- OCI image spec — [github.com/opencontainers/image-spec](https://github.com/opencontainers/image-spec)
