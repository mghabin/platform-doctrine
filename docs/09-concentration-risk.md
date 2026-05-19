# 09 — Concentration risk + Cloudflare contingency

> The recommended architecture concentrates DNS, CDN, WAF, edge compute, object storage, and admin access on **one vendor: Cloudflare**. That's an intentional bet — and bets need contingency plans.

## What's concentrated, and why it's a real risk

| Function | Vendor | Substitute if Cloudflare is down |
|---|---|---|
| Registrar | Cloudflare | Slow swap (transfer takes 5-7 days). Plan: keep registrar on Cloudflare; mitigate via DNS layer below. |
| Authoritative DNS | Cloudflare | **Secondary DNS** at AWS Route 53 or Google Cloud DNS (see below). |
| CDN + WAF + DDoS | Cloudflare | Azure Front Door as backup origin LB if Cloudflare is hard-down. Or direct-to-origin during a CF outage (degraded). |
| Edge compute (Workers) | Cloudflare | No fast substitute — re-deploy logic to ACA. Plan: keep Workers logic minimal and re-implementable on origin. |
| Object storage (R2) | Cloudflare | Azure Blob is the disaster-recovery backup target. Replicate continuously via lifecycle rules. |
| Zero Trust / admin access | Cloudflare | Tailscale or Azure Bastion as break-glass. |
| Email routing | Cloudflare | Direct MX to your transactional email provider (Resend / Postmark) as backup. |

Cloudflare has had outages — most recently a multi-hour July 2024 control-plane incident, and several hour-long edge incidents per year. The recommendation in [`01-recommendation.md`](./01-recommendation.md) treats Cloudflare as the *least* risky vendor in the stack (highest "low-issues" score), but that doesn't mean zero-risk. Plan for the day it's wrong.

## The single highest-leverage mitigation: secondary DNS

DNS is the chokepoint. If Cloudflare's DNS is down, your domains stop resolving — even if your origin is fine. **Secondary DNS** publishes the same zone to a second nameserver provider (e.g. AWS Route 53). Resolvers query whichever is up.

Cloudflare offers two modes:
- **Secondary DNS Setup** (Cloudflare as the primary, AXFR-out to a secondary like Route 53) — available; verify with the Cloudflare account team that AXFR-out is enabled on your zone before relying on it (it is not always enabled by default on Free/Pro and may require an explicit ticket).
- **Multi-provider DNS** (you publish to both providers from a single source of truth, e.g. via OctoDNS) — strongest pattern; requires more discipline.

The simpler "primary at Cloudflare + AWS as secondary" pattern is enough for the recommended stack. Configure during Phase 1 of [`05-phase-plan.md`](./05-phase-plan.md).

Cost: secondary DNS at AWS Route 53 ≈ $0.50/zone/month + $0.40 per million queries. For most orgs: <$5/month.

## DNSSEC + registrar lock (Phase 1)

- DNSSEC enabled at Cloudflare AND DS records published at registrar (Cloudflare Registrar does both ends automatically).
- Registrar lock on (default at Cloudflare Registrar). Verify.
- 2FA hardware-key required on the Cloudflare account.
- Auth-info code / transfer authorization stored offline.

## Account compromise — the real attack vector

Most "Cloudflare problems" aren't outages; they're account takeovers. Defenses:

| Defense | Setting |
|---|---|
| **Hardware 2FA only** | Cloudflare → Profile → Authentication → require WebAuthn. Disable TOTP fallback. |
| **Account audit log → SIEM** | Stream the [Cloudflare Audit Logs API](https://developers.cloudflare.com/api/operations/audit-logs-get-account-audit-logs) to Log Analytics for alerting. |
| **Scoped API tokens, not Global API Key** | Per-zone, per-permission, time-limited. Rotate quarterly. Never use the Global API Key in production. |
| **Config-as-code** | Manage zone records via OpenTofu Cloudflare provider. Lets you revert via `git revert` if the account is compromised. |
| **Out-of-band alerting** | Alert on DNS record changes / WAF rule changes via the audit log → Teams/PagerDuty. |

## What to do during a Cloudflare incident

### Origin reachability: pick ONE posture and design for it (resolves v1.1 contradiction with `07`)

The doctrine forces an explicit choice between two postures. **`07-origin-security-and-private-networking.md` recommends the security-first variant** for the prod default; the resilience-first variant is documented here for shops that explicitly trade origin reachability against degraded edge-only-failure mode.

| Posture | Prod origin reachability | Cloudflare edge-outage recovery | Trade-off |
|---|---|---|---|
| **Security-first (recommended)** | Internal-only ACA env + Cloudflare Tunnels — **no public IP on the origin** | Secondary DNS to a backup resolver + (if CF Workers/R2 are also down) accept extended outage until Cloudflare returns; **no grey-cloud bypass possible** | Maximum origin protection; tighter blast radius on account compromise; no DDoS bypass on edge outage |
| **Resilience-first** | Public ACA ingress restricted to Cloudflare IP ranges + AOP enforced | Same as above, *plus* emergency grey-cloud bypass: temporarily lift the AOP requirement, expose the ingress directly, accept that there's no WAF/CDN/DDoS until CF returns | Faster recovery on a pure edge outage at the cost of a wider always-on origin attack surface |

> Pick one. Document the choice in your platform-template repo. Drill the chosen failover quarterly. **Do not** silently rely on grey-cloud bypass while operating an internal-only origin — those are incompatible.

### If the CF edge is down — security-first variant (recommended)

1. Check [cloudflarestatus.com](https://www.cloudflarestatus.com/) and Cloudflare's incident ticker.
2. If you've set up secondary DNS (Phase 1), queries resolve via the backup nameserver — but the backup nameserver still points at the **same Cloudflare-tunneled origin**, which won't serve traffic if Workers/Pages/Tunnel control plane is degraded.
3. Surface a Cloudflare-incident-banner on a static "stale" copy of your marketing site (e.g. a Cloudflare Pages deployment of an emergency `503` page served from a separate CF account or from Bunny.net / Vercel as a backup edge — see `09-concentration-risk.md` § Cloudflare contingency).
4. Recover normally when CF is back.

### If the CF edge is down — resilience-first variant (if you chose it)

1. Check status page.
2. Temporarily remove the Cloudflare-IP-allowlist on the ACA ingress (`az containerapp ingress access-restriction remove …`) **and** disable AOP enforcement on the origin to allow direct browser-to-origin traffic.
3. Update DNS to point directly at the ACA FQDN (grey-cloud / unproxied).
4. Accept that you're operating without WAF/CDN/DDoS protection until CF returns.
5. When CF is back: re-enable AOP + IP allowlist FIRST, then re-proxy DNS.

### If Cloudflare account is compromised

1. **Containment**: revoke all API tokens via the dashboard (if still accessible) or call Cloudflare emergency support.
2. **Eviction**: rotate all session tokens, force re-auth on all team members, audit recent changes via the audit log.
3. **Recovery**: replay the OpenTofu config to restore the known-good zone state.
4. **Post-incident**: rotate all hardware 2FA keys; audit which secrets in Azure Key Vault might have been exfiltrated via the API token's scope.

### If Cloudflare goes fundamentally hostile (pricing / sanctions / acquisition)

1. **Trigger**: pricing change >2× expected, or a sanctions / export-control issue blocks your customer geographies, or an acquisition signals strategic-direction change you can't accept.
2. **Migration target**: secondary DNS provider takes over as primary. WAF/CDN moves to Azure Front Door (acceptable degradation) OR AWS CloudFront + AWS WAF.
3. **R2 migration**: lifecycle-replicate to Azure Blob; flip app config to point at Blob; tear down R2.
4. **Total elapsed**: ~4 weeks for a small org if practiced; ~3 months otherwise.

The key: **you can leave**. The cost-of-leaving is high enough that you won't do it casually, but bounded enough that Cloudflare can't extract rent indefinitely.

## Phase-by-phase concentration-risk gates

| Phase | Gate |
|---|---|
| Phase 1 | DNSSEC + registrar lock + secondary DNS provider configured |
| Phase 2 | Cloudflare zone managed via OpenTofu in a git repo (config-as-code) |
| Phase 3 | Audit logs streaming to LAW; hardware 2FA enforced on all team accounts |
| Phase 4 | Quarterly drill: simulate Cloudflare outage — verify secondary DNS resolves; verify direct-to-origin bypass works |
| Phase 5 | Documented Cloudflare account compromise runbook in `docs/runbooks/cf-compromise.md` |
| Phase 7 | Annual DR drill includes "Cloudflare hostile takeover" scenario |

## Cloudflare *can* be a single point of failure — that's a feature you bought, not a bug to fix

The reason the recommendation accepts the concentration: Cloudflare's blast radius for everything in the stack is bounded by **DNS + CDN**, not by application state. Application state lives on Azure. If CF is hard-down for 24 hours, the recovery is "swap DNS to the secondary, point traffic at the Azure origin, accept degraded WAF/CDN until CF returns".

That's different from concentration risk on a *stateful* vendor (e.g. all your data in Cosmos DB) — losing a stateful vendor is a multi-week migration; losing Cloudflare is a multi-hour failover.

Still, write the runbook. Practice the drill. Then sleep easier.

## Sources

- Cloudflare Secondary DNS — [developers.cloudflare.com/dns/zone-setups/zone-transfers/](https://developers.cloudflare.com/dns/zone-setups/zone-transfers/)
- Cloudflare Audit Logs API — [developers.cloudflare.com/api/operations/audit-logs-get-account-audit-logs](https://developers.cloudflare.com/api/operations/audit-logs-get-account-audit-logs)
- DNSSEC at Cloudflare — [developers.cloudflare.com/dns/dnssec/](https://developers.cloudflare.com/dns/dnssec/)
- OctoDNS (multi-provider) — [github.com/octodns/octodns](https://github.com/octodns/octodns)
- AWS Route 53 pricing (secondary DNS reference) — [aws.amazon.com/route53/pricing/](https://aws.amazon.com/route53/pricing/)
- Cloudflare Status page — [cloudflarestatus.com](https://www.cloudflarestatus.com/)
