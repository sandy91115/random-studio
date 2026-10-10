# Docker, VPS setup and resource planning

[Back to README](../README.md)

## Runtime decision

Use Docker locally and in CI to build/test OCI images; K3s runs those images through containerd in production. Do not run the same production apps/DBs through both Compose and Kubernetes. Compose-only production remains a valid alternative if you choose lower orchestration overhead. [Docker Compose production](https://docs.docker.com/compose/how-tos/production/)

## VPS assumptions and preflight

User-supplied configuration: 8 vCPU, 32 GB RAM, 400 GB NVMe, 32 TB bandwidth. Proposed OS Ubuntu 24.04 LTS. Verify from a provider console/SSH before installation:

```bash
# Read-only Linux checks
cat /etc/os-release
uname -m
nproc
free -h
df -h
lsblk
sudo ss -lntup
```

Confirm Linux amd64 vs arm64, real free resources, existing applications, mounted disks, public IPv4/IPv6, outbound DNS/HTTPS/SMTP rules, snapshots, provider firewall and rescue access. 32 TB is transfer allowance, not a throughput/latency or concurrency guarantee.

Never overwrite an existing web server or DB blindly. Traefik will need ports 80/443; plan migration of any current Nginx/Apache/docker proxy first.

## Starting memory budget

Planning allowances below are **not observed usage**, Kubernetes reservations or hard limits. Tune from measurements. Production starts with one replica per application.

| Component                           | RAM planning allowance |
| ----------------------------------- | ---------------------- |
| Ubuntu/system/cache baseline        | 3.00 GiB               |
| K3s control plane/system components | 2.00 GiB               |
| Ingress/TLS services                | 0.25 GiB               |
| Storefront + static admin           | 1.50 GiB               |
| API Gateway                         | 0.50 GiB               |
| Identity                            | 0.75 GiB               |
| Catalog                             | 1.00 GiB               |
| Commerce                            | 1.00 GiB               |
| Payment                             | 0.75 GiB               |
| Content                             | 0.50 GiB               |
| Notification                        | 0.50 GiB               |
| PostgreSQL                          | 4.00 GiB               |
| RabbitMQ                            | 1.50 GiB               |
| Redis                               | 0.50 GiB               |
| Monitoring/logging                  | 1.50 GiB               |
| **Total initial allowance**         | **19.75 GiB**          |

If the host reports 32 GiB usable, about 12.25 GiB remains for bursts, rollout overlap, optional staging and extra cache. Advertised GB and actual GiB differ; base decisions on `free -h`. A continuously running staging stack needs its own budget, initially around 4–6 GiB with minimal fixtures; reduce production/staging limits or move staging to another VPS if measured headroom is insufficient.

For Kubernetes, start total workload CPU requests conservatively around 4–5 cores and leave headroom for system/background/backup work; sum all actual manifests before deployment. Limits can exceed requests but do not create cores. Node scheduling uses allocatable capacity. App example: request 100m CPU/256Mi RAM; limit 1 CPU/768Mi RAM. Catalog/commerce/PostgreSQL need separate tuning. Do not assign 8 cores to every service.

Node heap cap must be below container memory limit with native/library overhead headroom. DB connections = replicas × pool size + migration/backup/admin connections. Start small pools (e.g. 5–10/app), validate against PostgreSQL limits and load, then consider PgBouncer if needed.

## Disk budget

Nominal 400 GB planning split, not partitions to create automatically:

| Use                         | Budget     |
| --------------------------- | ---------- |
| OS/K3s/system               | 30 GB      |
| Images and cached layers    | 40 GB      |
| PostgreSQL data/WAL         | 100 GB     |
| Local media, if selected    | 100 GB     |
| Temporary backup staging    | 40 GB      |
| Monitoring/logs             | 20 GB      |
| Free growth/safety capacity | 70 GB      |
| **Total**                   | **400 GB** |

Object storage frees local media allowance. Actual filesystem space is lower after formatting/reservations; alert on measured disk growth. 40 GB backup staging may become insufficient for full dumps; resize or stream safely to off-server storage. Never allow WAL/log/image retention to fill the root disk.

## Container build contract

Each app owns a Dockerfile. Build context is repository root so workspace packages/lockfile are accessible:

```bash
# Proposed: only after Dockerfile/app exists
docker build -f apps/catalog-service/Dockerfile -t random/catalog:dev .
```

Use multi-stage build: dependency/build stage → minimal non-root runtime. Keep Node/pnpm/base image versions pinned and periodically reviewed. [Docker multi-stage builds](https://docs.docker.com/build/building/multi-stage/)

| Build concern      | Required implementation                                                                                     |
| ------------------ | ----------------------------------------------------------------------------------------------------------- |
| Lockfile           | Frozen installation; no dependency upgrades during CI build                                                 |
| Workspace packages | Build/copy runtime dependencies with preserved resolution; verify pnpm deploy behavior for selected version |
| Runtime            | Production deps + verified dist entrypoint; no source bind mount                                            |
| Secrets            | Exclude env/keys/dumps with `.dockerignore`; use build secrets only when necessary                          |
| NestJS             | Bind `0.0.0.0:3000`; handle SIGTERM/shutdown; ESM imports and dist layout tested                            |
| Storefront         | Next.js standalone output + static/public assets; writable cache path when needed                           |
| Admin              | Build Vite assets, serve with non-root Nginx at 8080; SPA history fallback                                  |
| DB migrations      | Migration files + required CLI in an explicit migration target/image                                        |
| Health             | Small probes; don't assume curl exists in slim runtime                                                      |
| Logging            | stdout/stderr; redact secrets/PII and rotate at platform layer                                              |

Read-only root filesystem can be used after writable temp/cache paths are mounted. No privileged apps, host networking or Docker socket mounting. Add PID/resource limits and drop unnecessary capabilities.

## Image naming and release identity

```text
ghcr.io/<lowercase-owner>/random-studio-api-gateway:sha-<commit>
ghcr.io/<lowercase-owner>/random-studio-identity-service:sha-<commit>
ghcr.io/<lowercase-owner>/random-studio-catalog-service:sha-<commit>
... one image per app ...

Deployment reference:
ghcr.io/<owner>/random-studio-catalog-service@sha256:<digest>
```

Do not deploy `latest`. Record all app and migration image digests in the release bundle. One app change can deploy independently after compatibility checks; initially building all apps per release is simpler than unreliable path-based change detection.

## Local Compose design

Base `infra/compose/compose.yaml`: image/build definitions, service DNS, health checks and named volumes. Dev override: localhost published infra ports, safe fixtures and development commands. Production alternative override: digest images, restart policy, resource limits and only edge ports.

Networks: edge/frontends/gateway; backend owners; data connectivity scoped where practical. Docker network membership does not replace DB credentials or service authorization. Services calling OAuth/payment/email need controlled outbound connectivity; don't isolate all containers on an egress-blocked network accidentally.

Persistent storage for PostgreSQL/RabbitMQ; Redis persistence only if its use needs it. Never make Redis the only copy of a customer's order/cart intent you promise to preserve. Prefer DB-backed carts.

Compose health dependency ordering helps initial boot; apps still need runtime reconnect/retry behavior. Unhealthy containers are not automatically replaced solely because a Docker health check fails; restart policies act on process exits. Kubernetes probes provide different behavior.

## Host filesystem

```text
/opt/random-studio/
├── releases/<release-id>/             # Immutable config, digests, rendered manifests
├── state/{staging,production}.json    # Current/previous release pointers
├── scripts/                          # Reviewed deployment/backup tooling
└── compose/                          # Only if Compose production chosen

/etc/random-studio/
└── secrets/                          # Root-owned 0600, never committed

/srv/random-studio/
├── backup-staging/
├── restore-tests/
└── uploads/                          # Optional local-file strategy

/var/lib/rancher/k3s/                 # K3s/containerd/control-plane/local PV data
```

K3s local-path PVC data is provisioner-managed; don't invent `/srv/postgres` bind paths unless you explicitly configure that storage design. Named PVCs must outlive app releases. Raw live filesystem copies are not DB backups.

## Host bootstrap order

1. Snapshot/backup existing host, verify provider rescue access and disk/resources.
2. OS patches/time sync, SSH key account; confirm a second session before changing SSH rules.
3. Provider/host firewall with web ports public, administrative access restricted.
4. Create release/secret/backup locations with appropriate ownership.
5. Choose K3s production (follow [Kubernetes guide](kubernetes.md)) or Compose-only.
6. DNS/TLS, then infrastructure persistence/roles/backup validation.
7. Deploy staging, verify full workflows, then production promotion.

Docker Engine is needed for a Compose host; install from the official Ubuntu apt repository with Engine/Buildx/Compose plugin. Check existing conflicting packages before changes. K3s uses its own runtime and needs no separate Docker Engine. [Docker Ubuntu installation](https://docs.docker.com/engine/install/ubuntu/)

## Public ports

| Port                        | Policy                                                              |
| --------------------------- | ------------------------------------------------------------------- |
| 80/443 TCP                  | Public ingress only                                                 |
| SSH TCP                     | Key auth; restricted IP/VPN when feasible                           |
| 6443 Kubernetes API         | Admin/private access only; pipeline can use SSH on host             |
| 5432 / 5672 / 15672 / 6379  | Never public DB/broker/cache ports                                  |
| Internal NestJS/admin ports | ClusterIP/private Docker network only                               |
| K3s internode traffic       | Private trusted nodes only; requirements reviewed when adding nodes |

Audit both IPv4 and IPv6. Docker published ports may bypass UFW filtering; avoid publishing data ports, use provider firewall and Docker-aware filtering when Compose is selected. Verify externally, not only with host UFW status. [Docker firewall behavior](https://docs.docker.com/engine/network/packet-filtering-firewalls/)

## Compose-only production alternative

Choose this **instead of** K3s production. Require real image/secret/health/volume configuration first. Example Linux commands assume `/opt/random-studio/compose` contains implemented files and a protected non-secret image-release env file:

```bash
cd /opt/random-studio/compose
docker compose --env-file release.env -p random-production -f compose.yaml -f compose.production.yaml config --quiet
docker compose --env-file release.env -p random-production -f compose.yaml -f compose.production.yaml pull
# Run explicit one-off migration commands after compatible preflight/backup.
docker compose --env-file release.env -p random-production -f compose.yaml -f compose.production.yaml up -d --wait
docker compose --env-file release.env -p random-production -f compose.yaml -f compose.production.yaml ps
```

Use a deployment lock, same tested digests and previous release retention. Basic Compose recreation can interrupt traffic; no zero-downtime promise. Never use `down -v` for production updates. Do not run builds or self-hosted untrusted PR runners on the production VPS.
