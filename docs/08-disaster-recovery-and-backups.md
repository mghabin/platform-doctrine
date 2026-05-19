# 08 — Disaster recovery + backup playbook

> A backup never restored is Schrödinger's backup.

The phase plan ([`05-phase-plan.md`](./05-phase-plan.md)) introduces multi-region in Phase 4 and DR drills in Phase 7. This chapter is the operational playbook — what to back up, what the targets are, who's responsible, and how to prove it works.

> **Upstream doctrine.** This chapter applies, doesn't redefine, the reliability + data-state guidance in `mghabin/infra-engineering-guide`:
>
> - Tier 0/1/2/3 multi-region framework — `infra-engineering-guide/docs/08-data-state.md` §4 and `decision-trees.md` §6a
> - SLO + error-budget policy (a chapter *this* doctrine does not duplicate) — `infra-engineering-guide/docs/09-reliability.md` §1
> - Chaos-engineering maturity ladder — `infra-engineering-guide/docs/09-reliability.md` §8
> - Blameless post-mortem template — `infra-engineering-guide/docs/09-reliability.md` §4

> **Operator identity.** The commands in this chapter assume the operator runs them as the **CD Managed Identity** (Contributor on the workload resource group, Reader on shared platform RGs) — the same MI that GHA OIDC + FIC produces for deploys. For *break-glass* (the CD MI itself is compromised or revoked): use a pre-provisioned human emergency account with PIM-eligible Owner role on the subscription (activated through Entra PIM with MFA, expires in 1 hour). Never store a long-lived emergency password.

## RPO / RTO targets per tier

These are starting points — measure your actual values and adjust.

| Service tier (this doc) | Maps to infra-guide Tier | Example | RPO (data loss tolerance) | RTO (recovery time tolerance) | Pattern |
|---|---|---|---|---|---|
| Critical user-facing | **Tier 1** (cross-region async replica) | Auth, checkout | ≤ 5 min | ≤ 30 min | Active-passive multi-region + Postgres read replica |
| Standard user-facing | **Tier 2** (cross-region snapshot) | Most APIs | ≤ 1 hour | ≤ 4 hours | Single-region + 1 hour point-in-time restore |
| Internal admin | **Tier 3** (single-region daily backup) | Backoffice | ≤ 24 hours | ≤ 24 hours | Daily backup, manual restore |
| Analytics / batch | **Tier 3** (single-region weekly backup) | Reports | ≤ 7 days | ≤ 1 week | Weekly backup, rebuild from source |

> Mapping: this table uses the human-friendly tier names; the infra-engineering-guide chapter 08 §4 + decision-trees §6a uses **Tier 0/1/2/3** (0 = active-active; 1 = cross-region async replica; 2 = cross-region snapshot; 3 = single-region daily backup). Both are valid; pick the vocabulary your team already uses.

## What to back up

| Asset | Built-in backup? | Where it lives | Verify how often? |
|---|---|---|---|
| **Postgres data** | Yes — Flex Server PITR up to 35 days (7 by default; **extend to 35 days for prod under a compliance regime**) | Same region | Monthly restore test to scratch instance |
| **R2 object storage** | No automatic versioning by default | Cloudflare | Enable bucket versioning + lifecycle rules → monthly object-list integrity check |
| **Azure Blob** | Soft delete + versioning (opt-in) | Same region | Quarterly restore drill |
| **Key Vault secrets** | Soft delete + purge protection (must enable explicitly) | Same region | Annual purge-protection verification |
| **ACR container images** | Geo-replication available (Premium tier only) | Multi-region | Implicit in deploys; verify with `docker pull` from second region monthly |
| **Cloudflare config** | No native backup; export via API | n/a | Weekly export to git via cron Action — config-as-code |
| **GitHub repos** | GitHub's own DR; clone to a Hetzner box monthly | GitHub + your mirror | Monthly mirror sync test |
| **Bicep / OpenTofu state** | Azure Storage with versioning + soft delete | Same region | Annual state-restore drill |

## The Postgres failover playbook (multi-region)

The most critical and most-error-prone procedure. Drill it quarterly.

### Setup (one-time, in Phase 4 — multi-region)

1. Primary Postgres Flex in your `<primary-region>`, **General Purpose tier required** (Burstable doesn't support replicas; **also doesn't support PgBouncer** — both reasons to promote from Burstable before this Phase).
2. Read replica in `<secondary-region>`, async replication, **same legal geography as the primary if data-residency rules apply** (e.g. EU primary → EU secondary; see [`10-compliance-and-jurisdiction.md`](./10-compliance-and-jurisdiction.md)).
3. Verify replica lag in App Insights: typically < 1s for low-write workloads.
4. Promotion target FQDN baked into a Cloudflare DNS record with a separate name (e.g. `db-failover.internal.yourdomain.com`).
5. **For planned-switchover variant**: pre-assign a Reader Virtual Endpoint to the replica; `--promote-mode switchover` requires it.

> **Auth model assumption (resolves v1.1 contradiction with `01-recommendation.md`'s "no client secrets, ever"):** the application connects to Postgres via **Microsoft Entra authentication + Managed Identity** (no password in the connection string). Failover therefore rotates the **host/DNS only**, not a password-bearing secret. If your service still uses a password-based connection string (legacy or non-MI service), substitute step 4 with the secret-rotation variant noted inline below.

### Failover procedure (write this in `docs/runbooks/postgres-failover.md` and commit it)

```bash
# Trigger: primary Postgres in <primary-region> is unreachable for >5 min.

# Step 1 — Acknowledge alert in PagerDuty/Teams. Start the incident timer.

# Step 2 — Promote the secondary replica. Choose ONE variant:

# Variant 2a (EMERGENCY — primary confirmed dead, data-loss possible):
az postgres flexible-server replica promote \
  --name <replica-name> \
  --resource-group <rg-secondary> \
  --promote-mode standalone \
  --promote-option forced

# Variant 2b (PLANNED — graceful switchover, primary still reachable):
# Requires a Reader Virtual Endpoint pre-assigned to the replica in Phase 4 setup.
az postgres flexible-server replica promote \
  --name <replica-name> \
  --resource-group <rg-secondary> \
  --promote-mode switchover \
  --promote-option planned

# Step 3 — Verify the promoted server accepts writes:
psql "<connection-string>" -c \
  "CREATE TABLE _failover_test (id int); DROP TABLE _failover_test;"

# Step 4 — Update the application's DB hostname (MI auth path — no password rotation needed).
#   If your config has `Postgres__Host` as a separate setting from `Postgres__Connection`,
#   this is a single Key Vault secret update:
az keyvault secret set \
  --vault-name <kv-secondary> \
  --name Postgres--Host \
  --value "<promoted-server>.postgres.database.azure.com"

#   Legacy variant (password-based connection string — discouraged but supported):
#   az keyvault secret set --vault-name <kv-secondary> --name Postgres--Connection --value "<new-conn-string>"

# Step 5 — Force ACA to pick up the new secret. Two options:

# Option 5a (preferred — issue a new revision so the secret is re-read at startup):
az containerapp update \
  --name <app> \
  --resource-group <rg-secondary> \
  --set-env-vars FAILOVER_TS="$(date -u +%s)"

# Option 5b (restart specific revision — requires --revision name, not --name):
REVISION=$(az containerapp revision list \
  --name <app> \
  --resource-group <rg-secondary> \
  --query "[?properties.active].name | [0]" -o tsv)
az containerapp revision restart \
  --name <app> \
  --resource-group <rg-secondary> \
  --revision "$REVISION"

# Step 6 — Flip Cloudflare Load Balancer to the secondary region.
# NOTE: standard Cloudflare LBs are ZONE-scoped (not account-scoped):
curl -X PATCH "https://api.cloudflare.com/client/v4/zones/<zone-id>/load_balancers/<lb-id>" \
  -H "Authorization: Bearer $CF_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{"default_pools":["<secondary-pool-id>"]}'

# Step 7 — Smoke test the public endpoint:
curl -fsS https://api.yourdomain.com/health/ready

# Step 8 — Stop the incident timer. Record RTO actual.
```

**Post-incident**:

- `--promote-mode standalone` is **irreversible** — the old primary becomes a standalone server. `--promote-mode switchover` keeps the old primary as a new replica of the promoted one (planned-failover only).
- Provision a fresh replica in a third region for safety.
- Reverse-replicate (or rebuild) so the original primary region is a replica again, then plan failback.

## Restore verification (cron'd, not manual)

A backup is real only if you've restored from it in the last 30 days.

```yaml
# .github/workflows/restore-verify.yml — runs monthly
name: backup-restore-verify
on:
  schedule:
    - cron: '0 6 1 * *'   # 1st of every month, 06:00 UTC
  workflow_dispatch:

jobs:
  postgres-restore:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: azure/login@532459ea530d8321f2fb9bb10d1e0bcf23869a43 # v3.0.0
      - name: Restore prod backup to scratch instance
        run: |
          # 1. Find latest PITR target (24h ago)
          # 2. Restore to a new scratch server
          # 3. Connect + run integrity checks (row counts, key constraints)
          # 4. Drop the scratch instance
          # 5. Post result to Teams channel
  r2-integrity:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    steps:
      - name: Verify R2 object versions present
        run: |
          # Sample 100 random objects; verify version history exists
```

## Quarterly DR drill

| Quarter | Drill |
|---|---|
| Q1 | Postgres failover (above) — measure actual RTO |
| Q2 | Cloudflare config restore — wipe a zone in a scratch account, restore from git |
| Q3 | Full regional outage simulation — disable eastus in CF LB, run prod traffic on westus2 for 1 hour |
| Q4 | Key Vault soft-delete + restore drill; backup test of GitHub mirror |

After each drill: document actual times in `docs/runbooks/dr-history.md`. If actual RTO exceeds target, that's a Sev-2 follow-up.

## Compliance angle

For GDPR/SOC2/HIPAA, regulators care that:

1. You documented RPO/RTO targets (this doc).
2. You tested them (restore verification + DR drills).
3. You have evidence (`dr-history.md` + workflow run logs).

That's it. The procedures don't need to be exotic — they need to exist and be exercised.

## What this stack still doesn't protect against

- **Multi-region Azure outage** (rare but happened, e.g. May 2024 Entra cascade). Mitigation: design app to degrade gracefully when only the Cloudflare edge + R2 are reachable (read-only mode, cached content, queued writes for replay).
- **Cloudflare global outage**. Cloudflare *has* had them (July 2024 multi-hour). Mitigation: secondary DNS (NS records pointing at a backup provider — see [`09-concentration-risk.md`](./09-concentration-risk.md)).
- **Ransomware on backups**. Use Azure Blob immutability policies (legal-hold or time-based) + R2 versioning + separate-account backup destinations.

## Sources

- Postgres Flex backup + PITR — [learn.microsoft.com/azure/postgresql/flexible-server/concepts-backup-restore](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-backup-restore)
- Postgres Flex read replicas + promotion — [learn.microsoft.com/azure/postgresql/flexible-server/concepts-read-replicas](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-read-replicas)
- Key Vault soft-delete + purge protection — [learn.microsoft.com/azure/key-vault/general/soft-delete-overview](https://learn.microsoft.com/azure/key-vault/general/soft-delete-overview)
- R2 bucket versioning — [developers.cloudflare.com/r2/buckets/](https://developers.cloudflare.com/r2/buckets/)
- Cloudflare Load Balancing failover — [developers.cloudflare.com/load-balancing/](https://developers.cloudflare.com/load-balancing/)
- Google SRE Workbook — Disaster Recovery — [sre.google/workbook/](https://sre.google/workbook/)
