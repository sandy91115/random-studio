# Implementation roadmap and file structure

[Back to README](../README.md)

## Target file structure

This is the intended implementation, not a claim that these files exist. Braces denote separate directories.

```text
random_studio/
├── apps/
│   ├── storefront/
│   │   ├── src/app/
│   │   │   ├── page.tsx
│   │   │   ├── product/page.tsx
│   │   │   ├── product/[slug]/page.tsx
│   │   │   ├── product-category/[slug]/page.tsx
│   │   │   ├── cart/page.tsx
│   │   │   ├── checkout/page.tsx
│   │   │   └── account/
│   │   ├── src/features/{catalog,cart,checkout,account}/
│   │   ├── src/{components,lib}/
│   │   ├── public/
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── admin/
│   │   ├── src/app/{router,providers}/
│   │   ├── src/features/{auth,dashboard,products,categories,attributes}/
│   │   ├── src/features/{orders,customers,coupons,returns,content,settings}/
│   │   ├── src/{components,lib}/
│   │   ├── nginx.conf
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── api-gateway/
│   │   ├── src/{config,guards,filters,clients,routes,health}/
│   │   ├── src/{main.ts,app.module.ts}
│   │   ├── test/
│   │   ├── Dockerfile
│   │   └── package.json
│   ├── identity-service/src/modules/{auth,users,roles,addresses,sessions}/
│   ├── catalog-service/src/modules/{products,categories,attributes,reviews,inventory}/
│   ├── commerce-service/src/modules/{carts,coupons,checkout,orders,shipments,returns}/
│   ├── payment-service/src/modules/{payments,webhooks,refunds,reconciliation}/
│   ├── content-service/src/modules/{pages,banners,gallery,contact,enquiries,warranties}/
│   └── notification-worker/src/modules/{templates,delivery,retries}/
├── packages/
│   ├── contracts/src/{http,events}/
│   ├── api-client/src/
│   ├── ui/src/
│   ├── config/src/
│   ├── observability/src/
│   └── tsconfig/
├── tools/migration/
│   ├── src/{extract,transform,load,reconcile}/
│   ├── fixtures/
│   └── package.json
├── tests/{e2e,contracts}/
├── infra/
│   ├── docker/{admin,shared}/
│   ├── compose/
│   │   ├── compose.yaml
│   │   ├── compose.dev.yaml
│   │   ├── compose.production.yaml       # Alternative production path
│   │   └── postgres/init/
│   ├── k8s/
│   │   ├── platform/{cert-manager,monitoring}/
│   │   ├── infrastructure/{postgres,rabbitmq,redis}/
│   │   ├── base/{apps,network-policies}/
│   │   ├── overlays/{staging,production}/
│   │   └── jobs/{migrations,backups}/
│   └── scripts/
│       ├── bootstrap-vps.sh
│       ├── record-image.mjs
│       ├── render-release.mjs
│       ├── deploy-k3s.sh
│       ├── smoke-check.mjs
│       ├── rollback.sh
│       ├── backup-postgres.sh
│       └── restore-rehearsal.sh
├── .github/workflows/{ci,build-release,deploy-staging,promote-production}.yml
├── docs/
├── .dockerignore
├── .gitattributes
├── .gitignore
├── pnpm-workspace.yaml
├── pnpm-lock.yaml
└── package.json
```

Common domain-service structure:

```text
catalog-service/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── config/
│   ├── database/
│   ├── modules/products/
│   │   ├── dto/
│   │   ├── products.controller.ts
│   │   ├── products.service.ts
│   │   ├── products.repository.ts
│   │   └── products.module.ts
│   ├── messaging/{publishers,consumers,outbox}/
│   └── health/
├── prisma/{schema.prisma,migrations/}
├── test/
├── Dockerfile
├── .env.example
├── tsconfig.json
├── tsconfig.build.json
└── package.json
```

Controller handles DTO/transport/auth; service use cases; repository persistence. Add domain abstractions where checkout/payment invariants need them, not empty layers for simple CRUD.

## Monorepo setup

1. Preserve existing starter and Laravel reference; establish a reviewed initial Git commit. Current workspace files are untracked.
2. Move root Nest app into `apps/api-gateway`; update Nest/TS/Vitest configs/imports/build output.
3. Use pnpm workspace packages with their own app configs. Do not also introduce an overlapping Nest CLI monorepo build system.
4. Add React apps and scaffold domain services incrementally.
5. Shared packages contain contracts/config/UI/telemetry, not cross-service ORM/business models.
6. Pin supported Node runtime, pnpm version, action revisions and lockfile. Validate ESM compatibility.
7. Add Docker local infra and role-separated dev/test databases.
8. Add validation CI first; container/release workflows after real apps boot.

```yaml
# Proposed pnpm-workspace.yaml
packages:
  - apps/*
  - packages/*
  - tools/*
```

## Scripts to implement

Use package names `@random-studio/api-gateway`, `@random-studio/catalog-service`, etc.

| Script              | Contract                                            |
| ------------------- | --------------------------------------------------- |
| `dev`               | Run an individual app with watch mode               |
| `build`             | Compile app; workspace prerequisites built first    |
| `start:prod`        | Run verified emitted entrypoint                     |
| `lint`              | Lint actual app paths                               |
| `typecheck`         | Check TypeScript without stale output assumptions   |
| `test`              | Noninteractive unit tests                           |
| `test:integration`  | Real DB/broker checks in isolated environment       |
| `test:e2e`          | Root browser suite against disposable full stack    |
| `db:migrate:deploy` | Apply committed owner migrations, explicit Job only |
| `db:seed:dev`       | Dev fixtures; never production deployment step      |

Some scripts are new. Required CI checks must not silently skip through `--if-present`; explicitly scope apps/scripts instead.

## Local development workflow

Prerequisites: compatible Node/pnpm, Docker Desktop Linux containers/WSL2 on Windows, and implemented Compose/config scripts. XAMPP remains for the old Laravel reference.

```powershell
# Proposed commands; create the referenced files/scripts first.
pnpm.cmd install --frozen-lockfile
docker compose --env-file infra/compose/.env -f infra/compose/compose.yaml -f infra/compose/compose.dev.yaml up -d postgres redis rabbitmq
pnpm.cmd --filter @random-studio/identity-service run db:migrate:deploy
pnpm.cmd --filter @random-studio/catalog-service run db:migrate:deploy
pnpm.cmd --filter @random-studio/api-gateway run dev
```

Run other apps in separate terminals/root runner. Host Node apps use localhost infra mappings; container apps use service DNS. Provide separate env templates. Dev published DB/broker ports bind `127.0.0.1`. Test DBs never point to dev customer/production data.

## Roadmap

| Phase               | Estimate  | Deliverables                                                     | Acceptance gate                                             |
| ------------------- | --------- | ---------------------------------------------------------------- | ----------------------------------------------------------- |
| 0. Audit            | 1 week    | Existing/live routes, screens, data, integrations, parity sheet  | Working flows distinguished from placeholders               |
| 1. Foundation       | 1 week    | Workspace, Docker infra, config/logs/health/CI                   | Apps boot independently; role isolation verified            |
| 2. Identity         | 1–2 weeks | Login/register/OAuth/RBAC/profile/address/sessions               | Browser flows and forbidden-access checks pass              |
| 3. Catalog          | 2 weeks   | CRUD, variants, attributes, media, reviews, search, public pages | Admin-created product visible on desktop/mobile             |
| 4. Commerce         | 2–3 weeks | Guest merge, coupons, reservations, COD, account/admin orders    | COD flow; last-unit concurrency and duplicate requests pass |
| 5. Payments/events  | 1–2 weeks | Provider, verified webhooks, refunds, outbox/inbox, emails       | Sandbox success/failure/replay/late/timeout cases pass      |
| 6. Remaining parity | 2 weeks   | CMS/gallery/warranty/support/tracking/returns/admin              | Real browser CRUD/uploads/customer flows verified           |
| 7. Release/platform | 2–3 weeks | K3s staging, CI/CD, monitoring/backups, migration rehearsal      | Deploy, rollback and restoration exercised                  |

Platform work can overlap feature phases; rough solo range 12–18 full-time weeks, subject to learning/integrations/fidelity.

## Per-feature workflow

```text
Existing requirement → owner/schema/contract → use case + meaningful tests
  → React UI + typed client → browser/permissions/responsive checks
  → PR validation → staging verification → production promotion
```

First slice: admin login → category/product creation → listing → product detail. Next: guest cart → COD order → admin order view. Then online payment.

## Configuration

Each app validates its own required env keys. `.env.example` contains safe placeholders only. Extend `.gitignore` for nested env files, kubeconfigs, dumps, private keys, uploads and local secrets while allowing templates. `.gitattributes` should force LF for shell scripts on Windows.

| Group           | Configuration                                               |
| --------------- | ----------------------------------------------------------- |
| Common          | `NODE_ENV`, `PORT`, `LOG_LEVEL`, `SERVICE_NAME`             |
| Clients         | `IDENTITY_BASE_URL`, `CATALOG_BASE_URL`, `PAYMENT_BASE_URL` |
| Data            | Runtime `DATABASE_URL`; separate migration URL only in Jobs |
| Messaging/cache | `RABBITMQ_URL`, `REDIS_URL`                                 |
| Identity        | Signing keys, OAuth client credentials/callback             |
| Payment         | Provider credentials/webhook secret                         |
| Notification    | Email provider/SMTP credentials                             |
| Files           | Endpoint/bucket/owner-scoped object credentials             |
| Frontend        | Relative `/api/v1`; server-only internal gateway URL        |

Use same-origin `/api/v1` routing on each frontend host. Avoid baked staging API URLs so the exact tested image promotes unchanged. OAuth/webhook/public absolute URLs are environment config. Secrets never enter browser bundles.

## Verification

- Unit: money/coupon rules, state transitions, authorization decisions.
- Integration: stock atomicity, DB constraints/migrations, outbox/dedupe, signature fixtures.
- Contract: API/events remain compatible during rolling deployment.
- Browser: admin/public CRUD, guest merge, payments/COD, accounts/returns, responsive screens.
- Resilience: DB/broker/provider failures, process restart, duplicate/out-of-order messages, DLQ.
- Load: representative search/cart/checkout, p95 latency, pool use, CPU/RAM and queue age.

Build passing alone is not completion. Track verified/partial/untested status and clean up browser QA data.
