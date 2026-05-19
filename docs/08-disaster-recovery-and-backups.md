# 08 — Disaster recovery + backup playbook

> A backup never restored is Schrödinger's backup.

The phase plan ([`05-phase-plan.md`](./05-phase-plan.md)) introduces multi-region in Phase 4 and DR drills in Phase 7. This chapter is the operational playbook — what to back up, what the targets are, who's responsible, and how to prove it works.

## RPO / RTO targets per tier

These are starting points — measure your actual values and adjust.

| Service tier | Example | RPO (data loss tolerance) | RTO (recovery time tolerance) | Pattern |
|---|---|---|---|---|
| Critical user-facing | Auth, checkout | ≤ 5 min | ≤ 30 min | Active-passive multi-region + Postgres read replica |
| Standard user-facing | Most APIs | ≤ 1 hour | ≤ 4 hours | Single-region + 1 hour point-in-time restore |
| Internal admin | Backoffice | ≤ 24 hours | ≤ 24 hours | Daily backup, manual restore |
| Analytics / batch | Reports | ≤ 7 days | ≤ 1 week | Weekly backup, rebuild from source |

## What to back up

| Asset | Built-in backup? | Where it lives | Verify how often? |
|---|---|---|---|
| **Postgres data** | Yes — Flex Server PITR up to 35 days (7 by default) | Same region | Monthly restore test to scratch instance |
| **R2 object storage** | No automatic versioning by default | Cloudflare | Enable bucket versioning + lifecycle rules → monthly object-list integrity check |
| **Azure Blob** | Soft delete + versioning (opt-in) | Same region | Quarterly restore drill |
| **Key Vault secrets** | Soft delete + purge protection (must enable explicitly) | Same region | Annual purge-protection verification |
| **ACR container images** | Geo-replication available (Premium tier only) | Multi-region | Implicit in deploys; verify with `docker pull` from second region monthly |
| **Cloudflare config** | No native backup; export via API | n/a | Weekly export to git via cron Action — config-as-code |
| **GitHub repos** | GitHub's own DR; clone to a Hetzner box monthly | GitHub + your mirror | Monthly mirror sync test |
| **Bicep / OpenTofu state** | Azure Storage with versioning + soft delete | Same region | Annual state-restore drill |

## The Postgres failover playbook (multi-region)

The most critical and most-error-prone procedure. Drill it quarterly.

### Setup (one-time, in Phase 4)

1. Primary Postgres Flex in `eastus`, General Purpose tier (Burstable doesn't support replicas).
2. Read replica in `westus2`, async replication.
3. Verify replica lag in App Insights: typically < 1s for low-write workloads.
4. Promotion target FQDN baked into a Cloudflare DNS record with a separate name (e.g. `db-failover.internal.yourdomain.com`).

### Failover procedure (write this in `docs/runbooks/postgres-failover.md` and commit it)

```
Trigger: primary Postgres in eastus is unreachable for >5 min.

1. Acknowledge alert in PagerDuty/Teams. Start the incident timer.
2. Promote the westus2 replica:
   az postgres flexible-server replica promote --name <replica> --resource-group <rg> --promote-mode forced --promote-option planned
   (Use 'forced' only if primary is confirmed down; 'planned' for graceful.)
3. Verify the promoted server accepts writes:
   psql "<connection-string>" -c "CREATE TABLE _failover_test (id int); DROP TABLE _failover_test;"
4. Update the application connection string secret in Key Vault (westus2):
   az keyvault secret set --vault-name <kv-westus2> --name Postgres--Connection --value "<new-conn-string>"
5. Restart ACA services in westus2 to pick up the new secret:
   az containerapp revision restart --name <app> --resource-group <rg-westus2>
6. Flip Cloudflare Load Balancer to direct 100% traffic to westus2:
   curl -X PATCH "https://api.cloudflare.com/client/v4/accounts/<id>/load_balancers/<lb-id>" \
     -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
     --data '{"default_pools":["<westus2-pool-id>"]}'
7. Smoke test the public endpoint:
   curl -fsS https://api.yourdomain.com/health/ready
8. Stop the incident timer. Record RTO actual.

Post-incident:
- Promotion is irreversible — the old eastus primary is now standalone.
- Provision a NEW replica in a third region (centralus) for safety.
- Reverse-replicate westus2 → eastus once eastus is healthy, then plan failback.
```

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
      - uses: azure/login@<sha>
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
