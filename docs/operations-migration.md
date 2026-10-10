# Operations, recovery and Laravel migration

[Back to README](../README.md)

## Operational ownership

The single VPS is one failure domain. Kubernetes restarts apps; recovery from lost host/disk requires off-server backups and a reproducible bootstrap. Keep provider console access, DNS access, secrets recovery and incident instructions available outside this VPS.

Proposed targets, to validate through rehearsal:

| Target                      | Initial goal                                            |
| --------------------------- | ------------------------------------------------------- |
| App rollback                | Within 15 minutes for compatible releases               |
| Host recovery RTO           | Within 2–4 hours after resources/access are available   |
| DB RPO with daily dumps     | Up to 24 hours; generally too much loss for paid orders |
| DB RPO with tested WAL/PITR | Target ≤15 minutes, measured archive lag                |
| Media RPO                   | Based on object versioning/backup interval; document it |

Targets are not guarantees. Before taking real paid orders, enable/validate suitable database PITR or explicitly accept a larger data-loss window with a business recovery process.

## Monitoring and logs

Start with constrained metrics/alerts/log retention; proposed Prometheus + Grafana, with Loki optional after memory/disk planning. Dashboards stay private/admin-accessible. Do not store months of verbose logs on the same 400 GB disk.

| Signal              | Initial alert idea                                                                |
| ------------------- | --------------------------------------------------------------------------------- |
| Disk/PVC free       | Warning <25%, critical <15%; also monitor growth/WAL                              |
| Available RAM / OOM | Low headroom, OOMKilled, frequent restarts                                        |
| CPU                 | Sustained high use plus p95/error degradation                                     |
| HTTP                | 5xx increase, slow requests, failed external health checks                        |
| DB                  | Pool saturation, slow queries, deadlocks, replication/archive lag if configured   |
| RabbitMQ            | Oldest queue message age, retry/DLQ growth, disk/memory alarm                     |
| Outbox/workflows    | Old unsent outbox, stuck checkout/payment/refund states                           |
| Payments            | Signature failures, unmatched payment/order, amount mismatch                      |
| Backups             | Missing scheduled backup, failed upload, stale PITR archive, failed restore drill |
| TLS/DNS             | Renewal errors, approaching expiry, external route failure                        |

Tune thresholds to actual workload; a short spike is not automatically an incident. Alerts must reach a channel outside the failing VPS.

Log structured JSON with service/release/request/correlation IDs. Propagate trace context over HTTP/events. Redact passwords/tokens, full payment payloads, addresses and unnecessary personal data. Keep security/audit events separately with controlled retention. Avoid high-cardinality customer/order IDs in metric labels.

## Backups: separate layers

| Layer             | Backup method                                                                  | Recovery need                                              |
| ----------------- | ------------------------------------------------------------------------------ | ---------------------------------------------------------- |
| PostgreSQL        | Scheduled custom-format dumps + roles/config; production PITR/base backups/WAL | Owner data, schema and users                               |
| Media/invoices    | Object versioning + independent backup, or file-aware off-server copy          | Private/public file recovery                               |
| K3s control plane | Datastore-consistent backup + server token + config                            | Cluster/bootstrap secret state                             |
| RabbitMQ          | Definitions + deliberate durable-state recovery strategy                       | Users/vhosts/exchanges/queues and pending message handling |
| Redis             | Optional persistence/snapshot, depending on usage                              | Cache generally rebuilt; never only durable order source   |
| Releases/secrets  | Encrypted off-server config/digest/secrets backup                              | Reproduce known release on replacement host                |

Same-VPS backup directory is staging, **not disaster recovery**. Encrypt and upload backups to independent storage; monitor upload/checksum success. Set backup identity permissions deliberately; use versioning/immutability where available and separate recovery credentials.

Initial retention proposal: 7 daily + 4 weekly + 3 monthly logical recovery points, with separately sized PITR retention. Adapt to growth/retention requirements. Never rotate/delete the last known good backup before new backup/upload validation.

PostgreSQL supports logical dumps, filesystem-level approaches and continuous archiving; use database-aware tooling rather than raw live directory copies. [PostgreSQL backup overview](https://www.postgresql.org/docs/current/backup.html)

PITR needs compatible base backups and a continuous validated WAL archive. Archive lag/failure must alert; scheduled `pg_dump` alone is not PITR. Choose/test tooling such as pgBackRest or WAL-G before production and confirm version compatibility. [PostgreSQL PITR](https://www.postgresql.org/docs/current/continuous-archiving.html)

## PostgreSQL dump/restore example

**Illustrative administrative template only.** Requires `pg_dump`/`pg_restore`, a DB-reachable maintenance environment, securely supplied `PGHOST`/`PGUSER`, `.pgpass` or equivalent credentials, and authorized isolated restore DB. Avoid passwords in command arguments/logs. Use a dump client compatible with selected server major.

```bash
umask 077
mkdir -p /srv/random-studio/backup-staging
pg_dump --format=custom --no-owner --no-acl --dbname=random_catalog --file=/srv/random-studio/backup-staging/catalog.dump
# Upload encrypted backup + checksum to independent storage through backup tool.
```

Repeat for every owner DB with a manifest of timestamps/migration versions. Dump role definitions with a protected admin workflow; runtime users should not gain admin credentials. Cross-database dumps are not one atomic application-wide snapshot: for strict migration snapshots pause writes/outbox consumers and reconcile at the chosen boundary, or use an appropriate coordinated cluster backup strategy.

On a **separate restore environment** with an empty test database and required role created:

```bash
pg_restore --exit-on-error --no-owner --no-acl --dbname=random_catalog_restore /srv/random-studio/backup-staging/catalog.dump
```

Verify table counts/relationships/order totals/sample logins and app queries. A readable dump file is not proof of restoration. Never point restore rehearsal at production.

## K3s datastore recovery

Chosen initial datastore is SQLite. Back up it consistently using documented datastore procedure; use a cold maintenance copy or a validated consistent backup approach for live operation. Also back up `/var/lib/rancher/k3s/server/token`, encryption configuration and server configuration securely. Token is necessary to decrypt restored bootstrap data. Embedded etcd has a different snapshot workflow if later enabled. [K3s backup/restore](https://docs.k3s.io/datastore/backup-restore)

Cluster datastore backups **do not contain PostgreSQL PVC data or customer files**. Recovery must restore those independently. A YAML export is not sufficient protection for all secret/control-plane state.

## Broker recovery

Keep RabbitMQ definitions reproducible. Durable queue/PVC persistence handles ordinary restart, not disk loss. If pending messages are lost, outbox rows already marked sent will not automatically republish. Maintain replay/reconciliation records and an operator procedure, or implement validated broker-state backups/replication.

Replay uses original event IDs so consumers deduplicate. Reconcile payments directly with provider, compare commerce workflow state and resend only intended effects. Avoid duplicate emails/refunds during recovery. Never purge queues to clear an incident without understanding pending business work.

## Routine operations

- Daily: external health, orders/payment reconciliation, failed workflows/DLQ, backup freshness, disk growth.
- Weekly: dependency/security review, slow queries, capacity/pool limits, log/image retention.
- Monthly: off-server restore drill, TLS/secret rotation review and recovery timings.
- Planned maintenance: test OS/K3s/DB/broker upgrades in staging; snapshot/backup, announce necessary write freeze, serialize with deployments.

Use release-aware image garbage collection. Keep rollback digests in GHCR; do not blindly prune runtime volumes or manually remove files under containerd. Review inactive images/log retention with disk evidence.

## Incident workflow

```text
External alert → identify release/scope → inspect node/pod/DB/queue signals
  → preserve evidence → contain affected writes if needed
  → compatible app rollback OR forward fix OR data recovery
  → payment/order/event reconciliation → browser smoke → monitor
```

| Failure               | First response                                                                          |
| --------------------- | --------------------------------------------------------------------------------------- |
| One app CrashLoop/OOM | Inspect recent logs/events/resource limits/image/config; compatible rollback            |
| Readiness fails       | Inspect actual dependency/config; don't disable probe to hide issue                     |
| Broker unavailable    | Preserve outbox/backlog; reconnect, then idempotent replay                              |
| Provider timeout      | Reconcile provider status before retrying write/refund                                  |
| Disk full             | Stop growth source; assess WAL/log/images; free only verified safe data                 |
| DB unavailable        | Pause affected writes, diagnose storage/credentials/resources; avoid reset              |
| Host lost             | Replacement VPS → bootstrap → restore data/secrets/releases → reconcile → route traffic |

Read-only checks using an authorized kubectl identity:

```bash
kubectl -n random-production get pods,svc,pvc
kubectl -n random-production get events --sort-by=.lastTimestamp
kubectl -n random-production logs deployment/commerce-service --tail=100
kubectl -n random-production rollout status deployment/commerce-service --timeout=180s
```

Logs can contain sensitive information; control access and redaction. Record incident timeline/affected orders and validate actual customer flows after recovery.

## Laravel-to-new-system scope

Existing reference directory: `C:\xampp\htdocs\ramdonStudio`. Source routes show catalog/search/cart/coupons/checkout/accounts/Google login/admin/products/categories/attributes/returns/gallery/service settings/WooCommerce import/warranty/enquiries/static policies.

Route presence is not proof of functioning integration. Existing checkout source inspected in the conversation creates a pending PhonePe/COD order; verified provider payment must be implemented in the new service. WooCommerce import completeness is not established; verify present schema/runs/source records before relying on it.

**First choose authoritative source:** live WooCommerce, local Laravel DB, or a reconciled dataset. Do not assume local seeded/imported data represents current production customers/orders. Production provider/API access is not verified by this guide.

## Migration implementation

`tools/migration` owns restartable extract → transform → load → reconcile tools. Store each run's source timestamp/checksum, mappings and owner-level progress outside public APIs. Use secure source credentials and redact data logs.

| Source                                                 | Destination                                       |
| ------------------------------------------------------ | ------------------------------------------------- |
| Users/roles/addresses                                  | Identity                                          |
| Categories/products/attributes/variants/images/reviews | Catalog                                           |
| Orders/items/coupons/shipments/returns                 | Commerce                                          |
| Verified provider payment/refund history               | Payment; uncertain historical records flagged     |
| Pages/gallery/contact/enquiries/warranties             | Content                                           |
| Files                                                  | Object storage / chosen persistent media strategy |

Preserve slugs/SKUs/order numbers/source IDs and relationships. Transform MySQL-specific enums/defaults, decimal representations, timezones/booleans, HTML/assets and nullable fields intentionally. Inventory quantities need an authoritative stock reconciliation, not default fixture quantities.

Import idempotently using `(source_system, source_id)` mappings; use owner migration credentials/tools rather than cross-service SQL from runtime apps. Historical import suppresses normal outbox notifications/payment/refund side effects. Test restart mid-batch and duplicate source records.

Password formats require compatibility testing. Preserve compatible verified bcrypt hashes and rehash on login; unsupported formats require reset flow. Google provider IDs need explicit safe account mapping. Laravel sessions do not transfer automatically. Laravel encrypted service settings cannot be copied as ciphertext into Node; securely reconfigure credentials through the new secret mechanism.

## Reconciliation gates

- Counts by entity/status/source, duplicate/missing IDs and orphan references.
- Product SKU/slug/variant consistency; image counts/keys/checksums.
- Order item sums vs order totals, discounts/taxes/shipping and currencies.
- Customer/order ownership; address snapshots; fulfillment/tracking mappings.
- Payment totals/status vs provider evidence; refund history and unresolved records.
- Warranty invoice private access and enquiry/gallery/CMS completeness.
- Inventory physical/source balance; pending reservations and open orders.
- Representative old URLs, redirects, metadata and responsive visuals.

Keep a signed-off report of exceptions; do not silently coerce every unfamiliar status to paid/delivered.

## Cutover workflow

1. Rehearse full migration and restoration on staging with protected/anonymized data.
2. Lower DNS TTL in advance if DNS switching is needed; plan caches/payment callback URLs.
3. Verify target images, secrets, TLS, backups, monitoring and provider settings.
4. Schedule write freeze on old system; drain/pause relevant jobs and imports.
5. Capture final data boundary, export/import deltas, copy new files, reconcile.
6. Enable new production gateway/storefront; update routing/DNS and provider callbacks as planned.
7. Old site stays read-only/maintenance during cache propagation; prevent split writes.
8. Validate real production auth/catalog/admin/checkout/provider behavior with appropriate controlled transactions.
9. Reconcile pending old payments/orders/events across the boundary; start new workers.
10. Monitor, retain old backups/reference and remove migration fixtures/secrets when appropriate.

If DNS caches still hit old host, direct it safely to new traffic or show maintenance; don't accept orders in both independently. Plan handling of provider callbacks for transactions initiated on the old system before freeze.

## Cutover rollback boundary

Before new writes: reverting traffic to old system can be straightforward if old state is preserved. **After new orders/payments exist:** rollback needs write freeze plus export/reconciliation of new records, provider transaction mapping and event effects. Re-enabling the old database alone loses customer data.

Prefer a compatible app release rollback or forward fix over database rewind. Backup restoration is disaster recovery and may lose records since the recovery point; reconcile with provider and business records before reopening checkout.

## Launch acceptance checklist

- [ ] All services/images/config/migrations implemented and actual pipeline demonstrated.
- [ ] Public TLS/routing and private DB/broker/cache/API access verified externally.
- [ ] Customer/admin permissions and guest order privacy verified.
- [ ] Catalog/admin CRUD, desktop/mobile pages and uploads checked in browser.
- [ ] Guest merge, coupon, COD and verified online payment/refund flows pass.
- [ ] Duplicate checkout/events/webhooks and last-unit concurrency pass.
- [ ] Data migration counts/money/relationships/media reconciled.
- [ ] Old URLs/SEO/callback transitions verified.
- [ ] Off-server backup restore, app rollback and host recovery rehearsal completed.
- [ ] Queue/outbox/provider reconciliation and independent alert delivery work.
- [ ] Capacity measured under representative load; resource/retention budgets revised.

Check boxes only with evidence. Green builds and manifest validity alone do not establish production readiness.
