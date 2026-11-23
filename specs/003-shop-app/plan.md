# Implementation Plan: Shop Application (Requirements 2.1, 2.2, 2.3)

**Branch**: `003-shop-app` | **Date**: 2026-11-23 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/003-shop-app/spec.md`

## Summary

Two deliveries, strictly in order.

**Phase 1: shop-operator 0.2.0** (no app work before it is deployed). The Shop reconciler gains:

- a generated per-Shop admin password Secret passed as `ADMIN_PASSWORD`;
- for light shops, a generated Redis password Secret, wired into OT `Redis.redisSecret` and
  passed as `REDIS_PASSWORD`;
- Redis persistent storage with AOF;
- a pod sysctl so a non-root app can bind port 80;
- an automatic one-time recreate of 0.1.0 Redis instances that lack storage.

It ships as operator `v0.2.0`, chart `shop-operator` `0.3.0` and a kube-state PR. Live 0.1.0 shops
are upgraded by reconciliation alone.

**Then: the `shop` app.**

- One image: an ASP.NET Core (.NET 10) minimal API serving a built Angular app.
- Storage goes through one `IShopStore` abstraction with two implementations:
  - PostgreSQL: Npgsql, hand-written SQL, transactions with row locks.
  - Redis: StackExchange.Redis, Lua scripts for atomic changes.

  The implementation is chosen from the settings the operator injects.
- The admin signs in with the operator-generated password, then changes it. Sessions are cookies
  whose Data-Protection keys live in the store, so they work across replicas.
- Customers keep the cart in the browser. Placing an order atomically checks prices and stock,
  lowers stock and stores a `PendingPayment` order.
- One shared Testcontainers suite runs against both stores.
- CI mirrors shop-operator (`lint`, `test`, tag-driven `release` to GHCR). A final kube-state PR
  points the operator's Shop image at `ghcr.io/nikola031ue/shop:0.1.0`.

## Technical Context

**Language/Version**:

- Go (shop-operator, version from its `go.mod`).
- C# on .NET 10 LTS (SDK pinned in `global.json`).
- TypeScript with Angular (current stable major, pinned in `package-lock.json`); Node LTS for
  builds.
- Bash and YAML for charts, kube-state and CI.

**Primary Dependencies**:

- Operator: controller-runtime (existing).
- App: ASP.NET Core minimal APIs, `Npgsql`, `StackExchange.Redis`, and
  `Microsoft.Extensions.Identity.Core` (`PasswordHasher` only).
- Frontend: Angular router and HttpClient, no UI library.
- No ORM (research R7).

**Storage**: per Shop, PostgreSQL 17 (CNPG `Cluster`, `standard`) or Redis 7.2 (OT `Redis`,
`light`, now with password and 1 Gi PVC). Layouts are in [data-model.md](data-model.md).

**Testing**:

- Operator: `go test` (unit and envtest, existing harness).
- App: xUnit v3 with Testcontainers for .NET. One `ShopStoreContract` suite subclassed for
  PostgreSQL and Redis, plus API tests via `WebApplicationFactory` on both stores and unit tests.
- Frontend: `ng test` for the cart service and order-problem mapping.
- kube-state bats and e2e stay required.

**Target Platform**: Docker Desktop Kubernetes; image `linux/amd64`; CI `ubuntu-latest`.

**Project Type**: web application (Angular frontend and .NET backend in one image), plus operator
changes, one chart bump and two kube-state PRs.

**Performance Goals** (spec success criteria):

- Admin's first item within 2 min of Ready (SC-001).
- Order in under 1 min, visible to the admin within 5 s (SC-002).
- Search within 1 s at 1,000 items (SC-005).
- Image within 15 min of the tag (SC-008).

**Constraints**:

- Only the settings the operator injects (002 plus the 0.2.0 additions).
- Port 80, TCP readiness, 128 Mi memory limit.
- 2–3 replicas sharing nothing but the database.
- No secrets in git, logs or status.
- The 5.3 layout is immutable.
- Solo developer, about 25 tasks.

**Scale/Scope**: up to 1,000 items and 10,000 orders per shop. 4 repos touched: `shop-operator`,
`helm-charts` (version bump only), `kube-state` (2 PRs) and new code in `shop`.

All items are resolved in [research.md](research.md); no NEEDS CLARIFICATION remain.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.* Constitution v1.2.1.

| Principle | Check | Pre-research | Post-design |
|-----------|-------|--------------|-------------|
| I. Immutable IaC layout | Only a version bump of `charts/shop-operator`; `charts/shop` left as the placeholder (FR-026); kube-state changes stay inside `clusters/local/shop-operator/` | PASS | PASS |
| II. Repo-per-service, trunk-based | The app lives in its own repo `shop` (`shop-spec`). Branch protection before its first merge, squash-only, Conventional Commits, image `ghcr.io/nikola031ue/shop:X.Y.Z` from `vX.Y.Z`, no `latest`. The operator follows the same path as 002. The 002 merge commit `79cc773` in shophub-specs is a recorded one-off (spec Clarifications); merge commits are now disabled there | PASS (deviation recorded) | PASS (deviation recorded) |
| III. Tested before merge | The app has unit tests plus Testcontainers integration tests against real PostgreSQL **and** real Redis (one shared contract suite), required in CI. The operator changes have envtest coverage. Every bug fix gets a test | PASS | PASS |
| IV. Sepolia, tested locally | No chain interaction. The app only stores and shows the injected wallet address; payment is spec 005 | PASS | PASS |
| V. Platform in git, shops via API | Admin Secret, Redis Secret and the Redis PVC (through the Redis operator, `keepAfterDelete: false`) are derived resources owned by the Shop. No RBAC change. 0.1.0 shops are upgraded by reconciliation, not by hand. The image rollout goes through kube-state. Hand-applied test Shops in `qs` are tracked below, as in 002. The operator ClusterRole's cluster-wide Secret scope is unchanged (002 R-1, deferred to spec 004) | PASS (tracked) | PASS (tracked) |
| VI. Secrets in secret stores | Admin and Redis passwords are generated into Secrets and reach the app only via `secretKeyRef`. `REDIS_URL` stays credential-free. Pod annotations hold Secret UIDs, never values. The app logs no settings values, and tests assert that. Passwords are stored hashed (PBKDF2). CI uses `GITHUB_TOKEN` only | PASS | PASS |
| VII. Per-Shop observability | Still not provisioned; per-shop metrics, logs, traces, dashboards and alerts are spec 006. The app logs to stdout so 006 can collect it | DEVIATION (recorded) | DEVIATION (recorded) |

Technology constraints: Angular and .NET are in the constitution's list. Npgsql, StackExchange.Redis
and Testcontainers are libraries within the listed stack, so no justification is needed.

Gate result: **PASS**. Every exception is recorded in Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/003-shop-app/
├── plan.md              # this file
├── research.md          # R1–R15
├── data-model.md        # entities, PostgreSQL schema, Redis keys
├── quickstart.md        # validation scenarios 1–9
├── contracts/
│   ├── operator-0.2.0.md    # operator delta (Phase 1)
│   ├── http-api.md          # shop REST API
│   └── delivery.md          # repo layout, CI, release, rollout order
├── checklists/requirements.md
└── tasks.md             # /speckit-tasks
```

### Source Code

```text
shop-operator/                          # Phase 1 only
├── internal/resources/resources.go     # env refs, sysctl, annotations, Redis auth/storage/AOF
├── internal/resources/secrets.go       # new: admin / redis-auth Secret builders + random values
├── internal/controller/shop_controller.go   # ensure Secrets, RedisCredentialsLost, 0.1.0 recreate, Owns(Secret)
├── internal/controller/common.go       # reason constant
├── internal/controller/shop_controller_test.go, internal/resources/*_test.go
└── README.md                           # recovery notes (RedisCredentialsLost), env contract

helm-charts/charts/shop-operator/Chart.yaml   # 0.3.0 / appVersion 0.2.0 (sync-chart)
kube-state/clusters/local/shop-operator/helm.yaml    # 0.3.0          (PR 1)
kube-state/clusters/local/shop-operator/values.yaml  # shopImage → shop 0.1.0 (PR 2)

shop/                                   # new code, layout in contracts/delivery.md
├── src/Shop.Api/  (Program, Settings, Endpoints/, Auth/, Domain/, Storage/{IShopStore, Postgres/, Redis/})
├── src/Shop.Web/  (Angular: storefront, cart, checkout, admin login/items/orders/password)
├── tests/Shop.Api.Tests/ (Unit/, Integration/{ShopStoreContract, PostgresStoreTests, RedisStoreTests, ApiTests})
├── Dockerfile, README.md, .github/workflows/ci.yaml
```

**Structure Decision**:

- One backend project and one Angular project in the `shop` repo, plus one test project. Splitting
  into Domain, Infrastructure and Api assemblies adds nothing at this size.
- Storage is the single seam: `IShopStore` with `PostgresShopStore` and `RedisShopStore`
  (research R8).

## Delivery Notes (solo, ~25 tasks)

Estimated breakdown. `/speckit-tasks` will finalise it.

| Phase | Content | ≈ Tasks |
|-------|---------|---------|
| 1 Operator 0.2.0 | Secrets and env refs; Redis auth, storage and AOF; sysctl and annotations; 0.1.0 recreate and `RedisCredentialsLost`; envtest; README and 002 contract note; release `v0.2.0`; chart `0.3.0`; kube-state PR and live upgrade check (quickstart 1–2) | 6 |
| 2 App setup | Scaffold `shop` (solution, Angular, Dockerfile); CI workflow; branch protection (ask first) | 3 |
| 3 Foundation | Settings and startup init; `IShopStore` with the shared contract suite; PostgreSQL store; Redis store; admin auth (bootstrap, lockout, cookie, store-backed keys) | 5 |
| 4 US1 items | Admin item endpoints with tests; Angular admin (login, items, password) | 2 |
| 5 US2 order | Public items and order endpoints with concurrency tests; Angular storefront, cart and checkout | 2 |
| 6 US3 orders | Order list endpoint with tests; Angular orders page | 2 |
| 7 US4 release | Release `v0.1.0` and make the package public (ask first); kube-state rollout PR; quickstart 3–9 on the cluster | 3 |
| 8 Polish | `shop` README; self-review and secrets sweep | 2 |
| | **Total** | **≈ 25** |

Within budget (~25, hard stop at ~30), so no split of operator and app into separate specs is
needed.

Merges, GitHub settings changes and making a package public are each asked of the maintainer
first. Commits happen at the end of each phase, without pushing, per the standing rule.

## Complexity Tracking

| Violation / exception | Why Needed | Simpler Alternative Rejected Because |
|-----------------------|------------|--------------------------------------|
| Hand changes by the maintainer in throwaway namespace `qs` (Principle V): apply/delete test Shop, Wallet and DiscordChannel; delete a Secret and a Redis pod; restart pods (quickstart 2, 5, 8) | The ShopHub backend, the only allowed creator of user resources, is spec 004 | Waiting for 004 leaves the operator upgrade and the app unverifiable on a real cluster. Every hand change stays in `qs`, which is deleted afterwards; automated tests use envtest and Testcontainers |
| Per-Shop metrics, logs, traces, dashboards and alerts not provisioned (Principle VII) | Carried over from 002. Observability is spec 006 | Instrumenting before the observability stack exists targets nothing; the app logs to stdout and keeps the owner/shop labels 006 needs |
| Merge commit `79cc773` on shophub-specs `main` (Principle II linear history) | Spec 002 PR #1 was merged with a merge commit by mistake | Rewriting `main` needs a force-push on a shared branch. Accepted as a one-off; merge commits are now disabled in shophub-specs |
| Operator ClusterRole keeps cluster-wide Secret access (002 R-1) | 0.2.0 adds more Secrets of the same kind but no new rules | Narrowing it is planned in spec 004 together with the backend's namespaces |
| One-time data loss for 0.1.0 light shops when their Redis is recreated (research R4) | Storage can't be added to an existing StatefulSet | 0.1.0 light shops run only the nginx placeholder and have no persistent data, so nothing real is lost |
