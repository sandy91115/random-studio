# K3s Kubernetes production runbook

[Back to README](../README.md)

## Topology

One KVM 8 VPS runs one K3s server node that also schedules workloads. Use default SQLite cluster datastore initially, containerd runtime, Traefik ingress, CoreDNS and local-path storage. This topology has one failure domain; replicas on this node survive individual process failures, not host/disk failures.

K3s baseline requirements exclude workloads; actual application requests/limits must fit node allocatable capacity. [K3s requirements](https://docs.k3s.io/installation/requirements)

Production namespaces:

```text
random-production       apps + production data services/Jobs
random-staging          isolated apps + small staging data instances
monitoring              selected metrics/logging components
cert-manager            certificate controller
kube-system             K3s system services/Traefik
```

Separate staging credentials, PVCs, payment sandbox and email sinks. Namespaces need network policies and RBAC; alone they do not provide complete isolation. ResourceQuota/LimitRange keep staging from exhausting production headroom.

## Install sequence

No commands below have been executed against a VPS. Require verified OS/resources/firewall/port availability and backup/rescue access first.

1. Choose and pin a tested supported K3s release; record its Kubernetes version and upgrade policy. Avoid a floating install version.
2. Install on the server following [official installation guidance](https://docs.k3s.io/installation), using root-owned `/etc/rancher/k3s/config.yaml`.
3. Enable secret encryption at rest, protect API access, retain default Traefik for this design.
4. Verify node/system pods, DNS, storage provisioner and external ingress reachability.
5. Install pinned cert-manager components and configure Let's Encrypt **staging** issuer first; then production issuer after validation.
6. Create namespaces, quotas, network policies and initial service secrets through a secure bootstrap step.
7. Apply infrastructure manifests, wait for DB/broker/cache readiness, bootstrap owner roles/databases.
8. Apply application configuration, run migration Jobs, deploy apps and ingress.
9. Validate TLS, application health, browser workflows, backups and release rollback.

Example **configuration template**, not an executed installer:

```yaml
# /etc/rancher/k3s/config.yaml
write-kubeconfig-mode: '0600'
secrets-encryption: true
```

Configure node/system reservations based on measured host use before assigning workload requests. Protect `/etc/rancher/k3s/k3s.yaml`, cluster token and data. Do not commit kubeconfig or grant CI cluster-admin.

Read-only checks after installation:

```bash
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A
sudo k3s kubectl get storageclass
sudo k3s kubectl get svc -n kube-system
```

These administrative commands use the server's privileged kubeconfig; routine deployments use a scoped identity.

## Manifest map

| Component               | Resources                                                                         |
| ----------------------- | --------------------------------------------------------------------------------- |
| Each NestJS/Next.js app | Deployment, ClusterIP Service, ConfigMap, Secret references, service account      |
| Static admin            | Deployment + Service; non-root Nginx and SPA fallback                             |
| Notification worker     | Deployment + internal health Service; broker consumer                             |
| PostgreSQL              | Single-replica StatefulSet + stable Service + persistent PVC; app-owned DBs/roles |
| RabbitMQ                | Single-replica StatefulSet + Service/PVC; durable routes/queues                   |
| Redis                   | Single instance + Service; PVC only when persistence is required                  |
| Public routing          | Host/path Ingress + TLS Certificate/Issuer configuration                          |
| Migrations              | One-shot Job per owner/release with migration-only credentials                    |
| Backups                 | CronJob/off-host automation, protected credentials, monitored results             |
| Isolation               | Namespace quotas, NetworkPolicies, RBAC and restricted service accounts           |

Stateful infrastructure manifests are managed separately from normal app release rollouts. App deployment never replaces DB major versions, PVCs or broker storage automatically.

## Routing and TLS

Proposed hosts:

| Host/path                        | Destination                   |
| -------------------------------- | ----------------------------- |
| `randomstudio.in/`               | Storefront Service            |
| `randomstudio.in/api/v1/*`       | Gateway Service               |
| `admin.randomstudio.in/`         | Admin Service                 |
| `admin.randomstudio.in/api/v1/*` | Gateway Service               |
| Staging equivalents              | Services in staging namespace |

Admin and storefront use relative `/api/v1`. Preserve this prefix end-to-end if gateway defines it; do not also strip it via middleware. Next.js SSR uses an internal gateway DNS URL. OAuth callbacks/payment webhooks use public HTTPS routes through gateway.

Keep Traefik as the single edge owner; don't install a competing host Nginx on 80/443. K3s ships Traefik/ServiceLB; review exact release defaults and route config. [K3s networking services](https://docs.k3s.io/networking/networking-services)

Configure DNS A/AAAA only for working addresses, certificate ownership/renewal alerts, HTTP→HTTPS, correct proxy trust and rate/body limits. For payment webhooks preserve raw request body and provider-required headers. Apply CSRF bypass only to verified provider webhooks, not all API writes.

## App Deployment example

**Template only:** image digest, secrets, ConfigMap, Service and probes must exist. Replace the image placeholder before applying. App must expose these health paths on 3000 and support non-root execution. This is an app example, not a DB manifest.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog-service
  namespace: random-production
spec:
  replicas: 1
  revisionHistoryLimit: 3
  progressDeadlineSeconds: 180
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: catalog-service
  template:
    metadata:
      labels:
        app: catalog-service
    spec:
      automountServiceAccountToken: false
      terminationGracePeriodSeconds: 45
      imagePullSecrets:
        - name: ghcr-pull
      securityContext:
        runAsNonRoot: true
        runAsUser: 10001
        runAsGroup: 10001
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: app
          image: ghcr.io/replace-owner/random-studio-catalog-service@sha256:REPLACE_DIGEST
          ports:
            - name: http
              containerPort: 3000
          envFrom:
            - configMapRef:
                name: catalog-config
            - secretRef:
                name: catalog-runtime-secrets
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: '1'
              memory: 768Mi
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
          startupProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 5
            failureThreshold: 30
          readinessProbe:
            httpGet:
              path: /health/ready
              port: http
            periodSeconds: 5
            timeoutSeconds: 2
          livenessProbe:
            httpGet:
              path: /health/live
              port: http
            periodSeconds: 10
            timeoutSeconds: 2
```

Use readiness for capability to serve; liveness for process health. DB/broker outage should not trigger a restart storm through dependency-heavy liveness. Worker readiness can reflect broker connection; outbox-producing HTTP APIs may tolerate broker outage while DB stays available. Next.js/admin use their own health contract. [Kubernetes probes](https://kubernetes.io/docs/concepts/workloads/pods/probes/)

RollingUpdate with a surge pod requires spare resources and compatible old/new schemas/contracts. DB state and external side effects are not undone by Deployment rollback. [Kubernetes Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)

Graceful shutdown: stop readiness/new work, finish bounded in-flight requests, stop consumer intake, commit/ACK completed work, close pools. SIGTERM handlers and max request/job duration must fit termination grace.

## Storage

Start with local-path PVCs and deliberate ownership/retention policies. They persist pod restarts but remain tied to this server. Check provisioner reclaim policy, volume names, actual filesystem paths and recovery behavior before launch. Never delete PVCs during application rollout.

PostgreSQL data volume paths depend on the selected image major/version; use that image's documented mount location. Pin DB major version. A PostgreSQL major upgrade requires explicit migration/restore planning, not a changed image tag. DB/broker replicas remain one unless actual replication is configured.

Use object storage for media; local media alternative requires a dedicated persistent upload service/PVC. Scaling an upload app does not automatically share arbitrary host files across nodes. K3s datastore backup and PostgreSQL/PVC backups are separate requirements.

## Secrets and registry

- Bootstrap owner-scoped runtime secrets, separate migration secrets and read-only GHCR pull credentials securely.
- Base64 in a Kubernetes Secret is encoding; use encryption at rest, restrictive RBAC and protected backups.
- Never commit unencrypted Secret YAML, private kubeconfig or registry token.
- Prefer a secrets manager or encrypted configuration with controlled decryption for later automation.
- Use `imagePullSecrets` per namespace; isolate staging/production provider credentials.

[Kubernetes Secret practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)

## NetworkPolicy and RBAC

Default-deny application namespace traffic, then allow required paths: ingress controller→frontends/gateway; storefront SSR→gateway; gateway→owners; commerce→catalog/payment; owners→their DB/broker/cache; DNS and approved external provider egress. Test policies with real traffic; Kubernetes basic network policies do not filter provider domains by name.

Initial SSH CI model: deploy user invokes a narrowly reviewed host deployment wrapper using a namespace-scoped kubeconfig. Grant allowed app resources, Jobs and rollout reads; no PVC deletion, cluster roles or infrastructure modifications. Anyone allowed to create arbitrary pods can potentially mount namespace secrets, so treat deployment access as privileged and protect main/workflows accordingly.

No self-hosted runner on production for untrusted PRs. API 6443 remains private/admin-accessible. Bootstrap platform/CRDs/infrastructure separately with administrative access.

## Kustomize and release order

`base/apps` contains app manifests; overlays change namespace, hosts, resource limits, replicas and config. Release renderer pins all image digests and produces environment-specific manifests without secrets. Do not let routine app releases reapply destructive infrastructure changes.

```text
Platform bootstrap → namespaces/policies/secrets → stateful infrastructure
  → role/database bootstrap → migration Jobs → app Deployments/Services
  → ingress/TLS → rollout wait → smoke/browser checks
```

Commands **after manifests are implemented and migrations completed**:

```bash
# Run with the appropriate scoped kubectl context.
kubectl apply -k infra/k8s/overlays/staging
kubectl -n random-staging rollout status deployment/catalog-service --timeout=180s
kubectl -n random-staging get pods,svc,pvc
```

CI must wait for every changed app, not just catalog. A rollout timeout is failure requiring investigation; Kubernetes does not guarantee automatic rollback of the full multi-service release.

## Availability and growth

Start replicas at one; add two replicas selectively after testing graceful shutdown/shared sessions/idempotency and request budgets. HPA needs metrics, requests and tested min/max settings; on one node it can consume only remaining node capacity. Do not autoscale stateful DBs/brokers by replica count.

For server HA, add nodes and plan control-plane quorum, load balancing, PostgreSQL replication, broker availability and non-local persistent storage. Adding worker nodes alone does not make a single-server control plane or local PostgreSQL highly available.
