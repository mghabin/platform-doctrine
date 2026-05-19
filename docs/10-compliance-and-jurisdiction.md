# 10 — Compliance + jurisdiction

> The platform choice is also a regulatory choice. Cloudflare (US) + Azure (US-based, EU regions available) carries specific jurisdictional implications you need to be deliberate about.

This chapter covers GDPR, data residency, US-jurisdiction exposure, sanctions/export controls, and the certifications that matter for selling into regulated industries. It's not legal advice — get a lawyer for that — but it covers the platform-level decisions that should be made *before* you have customers and *can't* be retrofitted cheaply.

## The jurisdiction reality

Both Cloudflare and Microsoft Azure are **US-headquartered companies subject to US law**. This matters for three reasons:

1. **US CLOUD Act (2018)** — US authorities can compel a US-incorporated cloud provider to produce data stored anywhere globally. Even data physically located in EU regions can theoretically be subject to a US subpoena served on the provider's HQ.
2. **US export controls / sanctions (OFAC)** — the provider must refuse service to entities or individuals on US sanctions lists. If your customer base crosses geopolitical boundaries, this can affect you.
3. **Schrems II (2020)** — the CJEU invalidated the EU-US Privacy Shield. Cross-border transfers of EU personal data now require Standard Contractual Clauses (SCCs) + a Transfer Impact Assessment (TIA) showing that US surveillance law doesn't undermine the data subject's rights.

**You can still use US-headquartered providers for EU customer data** — most EU businesses do — but you need the paperwork (DPA + SCCs + TIA) and ideally the technical controls (encryption at rest with customer-managed keys, EU-region data residency, EU Data Boundary contracts).

## Data residency controls

### Azure

| Resource | Residency control |
|---|---|
| **Compute (ACA)** | Pick a region in the geography your customers' data must stay in. `westeurope` / `northeurope` / `germanywestcentral` for EU. |
| **Postgres Flex** | Same — region selection at create time. Backups stay in-region by default; geo-redundant backups (cross-region) are opt-in (avoid if EU-residency required). |
| **Storage (Blob)** | LRS (local), ZRS (zone), GRS (geo) options. EU-residency means LRS or ZRS within an EU region. |
| **Key Vault** | Region-pinned at create. |
| **Azure OpenAI** | Region-pinned. Microsoft publishes the [list of EU-region models](https://learn.microsoft.com/azure/ai-services/openai/concepts/models). For strict EU residency, use only EU-region deployments + don't enable "abuse monitoring" (which routes content to US for review). |
| **App Insights / LAW** | Region-pinned. **Watch out** — diagnostic settings can accidentally send data to a different region's workspace. |
| **EU Data Boundary** | Microsoft's contractual commitment that *most* customer data stays in EU regions for EU customers. Worth requesting in your EA. [learn.microsoft.com/privacy/eudb/](https://learn.microsoft.com/privacy/eudb/) |

### Cloudflare

| Resource | Residency control |
|---|---|
| **R2** | Pick a [jurisdiction (`eu`, `fedramp`)](https://developers.cloudflare.com/r2/buckets/data-location/#jurisdictional-restrictions) at bucket creation. Once set, data stays in that jurisdiction. |
| **Workers** | Cloudflare's [Regional Services](https://developers.cloudflare.com/data-localization/regional-services/) routes processing through specific regions (EU/US/India/etc.) — paid add-on. Worth it if you have EU customer requirements. |
| **D1** | Region-pinned at create. |
| **Logs** | Cloudflare Logpush can target a specific region's storage destination. |
| **Pages** | Static content is served from all 320+ POPs globally — no residency control on the *cached* copy. For strict EU-only delivery, use Cloudflare's Geo Restriction WAF rule. |

## Required paperwork (in order)

1. **DPA + SCCs** with Cloudflare and Microsoft. Both publish standard templates:
   - [Microsoft Online Services DPA](https://www.microsoft.com/licensing/docs/view/Microsoft-Products-and-Services-Data-Protection-Addendum-DPA)
   - [Cloudflare DPA](https://www.cloudflare.com/cloudflare-customer-dpa/)
2. **Transfer Impact Assessment** for each US-headquartered vendor handling EU personal data. Templates from the EDPB.
3. **Records of Processing Activities (ROPA)** — internal register required by GDPR Art. 30.
4. **Data Subject Rights playbook** — how you respond to access/deletion/portability requests. Postgres + Cloudflare R2 makes this tractable (single SQL query + S3-compatible delete API); some proprietary stores make it agonizing (avoiding which was a reason for the picks in [`01-recommendation.md`](./01-recommendation.md)).

## Customer-managed keys (CMK) — when you need them

Most B2C / SaaS apps don't need CMK. Some regulated industries do (finance, healthcare). When required:

- **Azure**: Key Vault with HSM-backed keys, CMK on Postgres + Storage + Key Vault itself.
- **Azure OpenAI**: CMK supported on the resource, not on the prompts/completions (those are encrypted with platform keys).
- **Cloudflare R2**: server-side encryption with customer-provided keys (SSE-C) — available; per-request key.

Adds operational complexity (key rotation, key escrow, break-glass). Don't enable until you have a compliance requirement that demands it.

## Compliance certifications worth knowing

| Cert | What it means | Coverage in this stack |
|---|---|---|
| **SOC 2 Type II** | Operational controls audited annually | Azure + Cloudflare both have SOC 2 Type II reports — request them via the trust portal |
| **ISO 27001 / 27017 / 27018** | International security mgmt standards | Both providers covered |
| **HIPAA** | US healthcare PHI handling | Azure: covered (sign BAA). Cloudflare: covered for some services (R2 + Workers with HIPAA-eligible config); check before storing PHI |
| **PCI-DSS** | Payment card data | Don't store card data — use Stripe / Adyen and pass tokens. Cloudflare + Azure can host PCI workloads but it's a major scope expansion |
| **FedRAMP** | US Federal cloud authorization | Azure Government region available; Cloudflare `fedramp` jurisdiction available. Don't pursue unless you have a federal contract |
| **GDPR Art. 28 (processor obligations)** | EU data processing | DPA + SCCs cover this |
| **EU AI Act** | High-risk AI systems | Azure OpenAI's enterprise governance + content filter + audit logs help; document model usage |

## Sanctions / export-controls exposure

Both Cloudflare and Microsoft enforce OFAC / EU sanctions. Practical implications:

- **Your customers in sanctioned jurisdictions can't use your service** even if the legal entity is in a permitted country. If you sell into emerging markets, run a quarterly customer-list check against the OFAC SDN list.
- **Your team in sanctioned jurisdictions can't be granted cloud access**. This affects hiring.
- **GitHub also enforces sanctions** — affects code-hosting choice.
- **The list changes**. Be prepared to lose access to a region within days of a sanctions designation.

Mitigation: never bet the business on a single sanctioned-risk jurisdiction. Diversify customer geography. Avoid making vendor choices that are *worse* on sanctions than the providers you've already accepted.

## When the recommendation breaks for compliance reasons

This entire doctrine assumes a workload that can run on US-headquartered providers. If your situation is one of these, **this doctrine is wrong for you** and you need a different one:

- **EU-sovereign-only requirements** (some German/French government contracts, some defense work): use OVHcloud / Scaleway / Exoscale instead of Azure. Cloudflare can stay if "data plane only in EU" is acceptable; otherwise use Fastly EU presence or Bunny.net.
- **Chinese-market-only**: Azure China (operated by 21Vianet) or Alibaba Cloud. Cloudflare China is via JD Cloud partnership and limited.
- **Pure-air-gapped / classified**: not addressed by this doctrine at all.

For the 90% case (commercial SaaS, B2C, B2B with mixed geography), Cloudflare + Azure with EU-region deployments + the paperwork above is fine. Just be deliberate.

## Compliance gates by phase

| Phase | Compliance gate |
|---|---|
| Phase 1 | DPA signed with Cloudflare and Microsoft; SCCs in place if any EU customer planned |
| Phase 2 | Azure tags include `dataClassification`; Azure Policy enforces region restriction per workload |
| Phase 3 | Privacy policy + cookie banner before customer signup |
| Phase 4 | TIA documented for any cross-border data flow |
| Phase 5 | ROPA maintained automatically (e.g. via DataGrail / OneTrust integration) |
| Phase 7 | Annual SOC 2 audit (if selling to enterprises) |

## Sources

- Microsoft Trust Center — [microsoft.com/trust-center](https://www.microsoft.com/trust-center)
- Cloudflare Trust Hub — [cloudflare.com/trust-hub/](https://www.cloudflare.com/trust-hub/)
- EU Data Boundary — [learn.microsoft.com/privacy/eudb/](https://learn.microsoft.com/privacy/eudb/)
- Schrems II decision summary (EDPB) — [edpb.europa.eu/our-work-tools/our-documents/recommendations/recommendations-012020-measures-supplement-transfer_en](https://edpb.europa.eu/our-work-tools/our-documents/recommendations/recommendations-012020-measures-supplement-transfer_en)
- Cloudflare R2 jurisdictional restrictions — [developers.cloudflare.com/r2/buckets/data-location/#jurisdictional-restrictions](https://developers.cloudflare.com/r2/buckets/data-location/#jurisdictional-restrictions)
- Cloudflare Regional Services — [developers.cloudflare.com/data-localization/regional-services/](https://developers.cloudflare.com/data-localization/regional-services/)
- US CLOUD Act overview (CRS report) — [crsreports.congress.gov/product/pdf/LSB/LSB10125](https://crsreports.congress.gov/product/pdf/LSB/LSB10125)
