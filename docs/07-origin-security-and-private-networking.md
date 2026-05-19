# 07 — Origin security + private networking

> If your Azure origin is reachable from the public internet, bypassing Cloudflare is one `nslookup` away. Lock the path.

The architecture in [`01-recommendation.md`](./01-recommendation.md) puts Cloudflare in front of Azure Container Apps as the public-facing edge. By default, the ACA ingress FQDN is also publicly reachable — anyone who discovers it can bypass Cloudflare's WAF, rate limiting, and bot protection. This chapter covers the controls that close that path.

## The three concerns

1. **Origin discoverability** — DNS/CT logs leak the ACA FQDN.
2. **Origin reachability** — even if hidden, the ACA ingress accepts any request that knows the hostname.
3. **Lateral movement on the Azure side** — if an attacker reaches one service, they need to be contained from reaching Postgres / Key Vault / Storage.

## Defenses, by Phase

### Phase 1 (foundation) — turn these on at day 1

- **Cloudflare proxied DNS** (`orange-cloud` on every public record). Cloudflare returns its own IPs; the origin IP is hidden from public DNS.
- **HSTS** with `preload`, `includeSubDomains`. Cloudflare adds this automatically when SSL=Full(strict). Submit to the [HSTS preload list](https://hstspreload.org/) once you're confident.
- **DNSSEC** enabled at Cloudflare (free) and at your registrar (Cloudflare Registrar handles both ends automatically).
- **Registrar lock** — Cloudflare Registrar is locked by default; verify it's on.

### Phase 2 (platform template) — bake these into every new service

- **Cloudflare Authenticated Origin Pulls (AOP)** — origin only accepts requests carrying a Cloudflare client cert. Free on all Cloudflare plans. ACA can verify the cert at the ingress via custom `X-Forwarded-*` rules, or via a thin reverse-proxy container if you need stricter validation.
- **Origin allowlist by Cloudflare IP ranges** — ACA ingress can restrict source IPs. Set the allowlist to Cloudflare's published [IPv4](https://www.cloudflare.com/ips-v4) + [IPv6](https://www.cloudflare.com/ips-v6) ranges. Update via cron (CF publishes a versioned JSON; check monthly).
- **Custom hostname only** — don't expose the `*.azurecontainerapps.io` FQDN publicly; use only `api.yourdomain.com`. Document that the platform FQDN is internal-only.

### Phase 3+ (after first product) — private networking, properly

- **ACA Environment with VNet integration** ([learn.microsoft.com/azure/container-apps/networking](https://learn.microsoft.com/azure/container-apps/networking)). Choose `internal` env type — ingress is only reachable from inside the VNet.
- **Cloudflare Tunnels** (`cloudflared`) — bridges Cloudflare's edge to the internal ACA env without opening any inbound port on Azure. The tunnel container runs as a sidecar in ACA, makes outbound-only connections to Cloudflare. Free. **This is the cleanest pattern**: zero public IPs on Azure.
- **Private Endpoints** for stateful services:
  - **Postgres Flexible Server** → Private Endpoint in the ACA VNet. Disable public network access entirely.
  - **Key Vault** → Private Endpoint. Disable public network access (Defender for Cloud will flag this as compliant).
  - **Storage Account** → Private Endpoint per blob/file/queue/table service.
  - **Service Bus** → Private Endpoint (Premium tier required — note the cost step).
- **Outbound egress control**:
  - ACA in `internal` env routes outbound via the VNet — pair with **Azure NAT Gateway** for predictable egress IPs (which you can then allowlist on third-party APIs).
  - Alternative: **Azure Firewall Basic** (~$0.40/hr ≈ $290/mo) if you need centralized egress filtering. Skip on small orgs.
- **Network Security Groups** on the VNet subnet — deny-by-default; allow only the specific subnets that need to reach each PaaS Private Endpoint.

## What this stack still doesn't protect against

- **Compromised Cloudflare API token** → attacker turns off proxying. Mitigation: short-lived API tokens scoped to a single zone, MFA on the CF account, audit logs to SIEM.
- **Compromised Azure RBAC role** → attacker re-opens public access. Mitigation: Azure Policy enforcement (`Microsoft.KeyVault/vaults` deny `publicNetworkAccess=Enabled` at the management-group level).
- **DDoS at L7 that costs you in CF egress beyond plan** → Cloudflare Pro/Business plans absorb most; Enterprise tier has bandwidth pricing escalators. Set Cloudflare's Workers rate-limiting + bot-fight-mode early.

## Origin security checklist (gate Phase 3 on this)

- [ ] All public DNS records `proxied` (orange-cloud) at Cloudflare
- [ ] SSL/TLS = Full (strict)
- [ ] HSTS enabled + preload submission scheduled
- [ ] DNSSEC enabled at registrar + zone
- [ ] Cloudflare Authenticated Origin Pulls enabled OR Cloudflare Tunnels deployed
- [ ] ACA ingress IP allowlist = Cloudflare IP ranges (or `internal` env with Tunnels)
- [ ] Postgres / Key Vault / Storage have `publicNetworkAccess: Disabled` in prod
- [ ] Azure Policy at mgmt-group level deny-creating PaaS resources with public access enabled
- [ ] NSG on every subnet: deny-by-default with explicit allow rules
- [ ] Quarterly origin-discovery dry-run: try to reach the ACA FQDN from outside the VNet — must fail

## Sources

- ACA networking — [learn.microsoft.com/azure/container-apps/networking](https://learn.microsoft.com/azure/container-apps/networking)
- Cloudflare Authenticated Origin Pulls — [developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/](https://developers.cloudflare.com/ssl/origin-configuration/authenticated-origin-pull/)
- Cloudflare Tunnels — [developers.cloudflare.com/cloudflare-one/connections/connect-networks/](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- Cloudflare IP ranges (programmatic) — [api.cloudflare.com/client/v4/ips](https://api.cloudflare.com/client/v4/ips)
- Azure Private Link overview — [learn.microsoft.com/azure/private-link/private-link-overview](https://learn.microsoft.com/azure/private-link/private-link-overview)
- HSTS Preload List — [hstspreload.org](https://hstspreload.org/)
