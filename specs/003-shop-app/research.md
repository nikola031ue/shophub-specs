# Research: Shop Application (003)

Each entry: Decision / Rationale / Alternatives considered. Operator items (R1–R6) are delivered
in Phase 1, before any app work.

## R1 — Admin initial password (FR-004)

- **Decision**: The Shop reconciler creates Secret `<shop>-admin` (key `password`) once, with 24
  random URL-safe characters from `crypto/rand` (~143 bits), labelled
  `app.kubernetes.io/managed-by=shop-operator` plus the Shop's owner labels, controller-owned by
  the Shop. The Deployment gets env `ADMIN_PASSWORD` via `secretKeyRef`. Existence is checked
  through the cache and confirmed with the uncached `APIReader` before creating, as for Wallet
  keys in 002. If the Secret is deleted, the operator generates a new one.
- **Rationale**: Clarification Q1=A. A derived resource owned by the Shop (Constitution V). The
  existing label-filtered Secret cache sees it, so no RBAC change. Regenerating is safe because of
  R11: the app only uses the value while the admin still has the generated password.
- **Alternatives**: password in Shop status (forbidden, VI); app-generated setup code (rejected in
  Q1); a hash on the Shop resource (CRD change, rejected in Q1).

## R2 — Redis authentication (FR-020, closes 002 R-3)

- **Decision**: For `light` Shops the reconciler creates Secret `<shop>-redis-auth` (key
  `password`, 32 random chars) once and sets OT `Redis`
  `spec.kubernetesConfig.redisSecret: {name: <shop>-redis-auth, key: password}`. The app gets env
  `REDIS_PASSWORD` via `secretKeyRef`; `REDIS_URL` stays credential-free
  (`redis://<shop>-redis.<ns>.svc:6379`). In 0.2.0, if the Secret disappears while the Redis
  exists, the operator does **not** generate a new one. The Shop reports `Ready=False`, reason
  `RedisCredentialsLost`. Recovery is to delete and recreate the Shop (documented like 002's
  `KeyLost`).
- **Update 2026-11-24, operator 0.2.1 (T026)**: a lost Secret is regenerated under the next name
  (`<shop>-redis-auth-<n>`) and the Redis CR's `redisSecret.name` is switched to it. The OT
  redis-operator v0.27.0 rolls the StatefulSet on that change and the PVC data survives (checked
  live with a throwaway Redis in `qs`, then in the `lite` drill). This was the maintainer's
  preferred path over pod-delete RBAC, which was not needed. `RedisCredentialsLost` is retired.
- **Rationale**: `redisSecret` is in the vendored OT v1beta2 CRD (`test/crds/`). A new password
  would need a Redis restart, and the operator has no pod permissions. Refusing to drift is
  simpler and safe.
- **Alternatives**: password inside `REDIS_URL` (puts a secret into a plain env value); ACL users
  via `spec.acl` (more moving parts for one client); NetworkPolicies only (doesn't authenticate;
  Docker Desktop doesn't enforce them).

## R3 — Redis persistence (FR-021)

- **Decision**: Set `spec.storage.volumeClaimTemplate.spec` to `ReadWriteOnce`, `1Gi`, default
  StorageClass (`hostpath` on Docker Desktop). Set `spec.storage.keepAfterDelete: false`, so the
  Redis operator removes the PVC when the Redis is deleted, and that delete cascades from the
  Shop. Set `spec.redisConfig.dynamicConfig: ["appendonly yes"]`, so writes survive a crash, not
  only a clean shutdown. Quickstart checks `CONFIG GET appendonly` and that data survives a Redis
  pod delete.
- **Rationale**: Clarification Q3=A. The PVC's lifecycle stays with the Redis operator, so the
  shop-operator needs no PVC permissions, and deleting the Shop still removes the data (SC-003 of
  002 is kept).
- **Alternatives**: `persistentVolumeClaimRetentionPolicy` (same effect, but StatefulSet-level and
  less explicit); RDB snapshots only (loses up to an hour of writes on a crash).

## R4 — Upgrading 0.1.0 Shops automatically (spec Dependencies)

- **Decision**: Reconciliation does the upgrade; nobody touches Shops by hand.
  - Admin Secret, Redis Secret and the new env vars are added by the normal apply. The changed
    pod template rolls the shop pods.
  - A live OT `Redis` without `spec.storage` was created by 0.1.0. StatefulSet
    `volumeClaimTemplates` are immutable, so the reconciler **deletes** that Redis once, logs it
    (no Kubernetes event, so no RBAC change), and creates the new one on the next pass.
  - This loses 0.1.0 light-shop data, but there is none: 0.1.0 runs the nginx placeholder and has
    no persistent Redis.
- **Rationale**: The only way to add storage to an existing StatefulSet. Deciding on `spec.storage`
  being absent makes it a one-time action that never fires again.
- **Alternatives**: in-place update (the Redis operator fails on the immutable field); asking the
  maintainer to recreate Shops (a hand change, Constitution V).

## R5 — Shop pods may bind port 80 as non-root

- **Decision**: Operator 0.2.0 sets pod `securityContext.sysctls: [{name:
  net.ipv4.ip_unprivileged_port_start, value: "0"}]` on the shop Deployment. The port stays 80,
  as in the 002 contract.
- **Rationale**: The app image runs as a non-root user (R13) and listens on the operator's port 80.
  This sysctl has been in Kubernetes' safe set since 1.22, so no PodSecurity or kubelet flag is
  needed. The nginx placeholder is unaffected. The operator is being released anyway (R1–R4).
- **Alternatives**: a configurable port in the operator (new chart value, CRD-adjacent contract
  change); running the app as root (weaker); `NET_BIND_SERVICE` capability (chiseled images
  don't honour file capabilities for dotnet).

## R6 — Operator release path

- **Decision**: Operator `v0.2.0` tag → image `ghcr.io/nikola031ue/shop-operator:0.2.0`.
  helm-charts `shop-operator` `0.3.0`, appVersion `0.2.0`; `sync-chart.sh` regenerates CRDs and
  RBAC. Neither changes, because the API is unchanged and Secret verbs already exist. One
  kube-state PR bumps `clusters/local/shop-operator/helm.yaml` to `0.3.0`. Each step is its own PR
  with green CI.
- **Rationale**: The same path proven in 002 (T017–T023). The Shop CRD is unchanged, so this is a
  MINOR release.

## R7 — Backend stack

- **Decision**: .NET 10 (LTS) ASP.NET Core minimal APIs in one project `Shop.Api`. `Npgsql` for
  PostgreSQL with hand-written SQL, `StackExchange.Redis` for Redis, `PasswordHasher<T>` from
  `Microsoft.Extensions.Identity.Core` for hashing. No EF Core, no Dapper, no MediatR. SDK pinned
  in `global.json`; packages pinned by exact version with a lock file.
- **Rationale**: User input: "no ORM-heavy layers". Ten queries don't need an ORM. .NET 10 is the
  supported LTS at planning time.
- **Alternatives**: EF Core (heavy, and doesn't cover Redis anyway); Dapper (small gain over plain
  Npgsql for ~10 queries).

## R8 — One repository abstraction, two implementations

- **Decision**: One interface, `IShopStore`, covering items, orders, the admin record,
  Data-Protection keys and `InitializeAsync` (schema setup under a lock).
  - Implementations: `PostgresShopStore` and `RedisShopStore`, chosen at start from the injected
    settings (R10).
  - One abstract xUnit test class `ShopStoreContract` holds every behavioural test. It is
    subclassed once per store, each with its Testcontainers container, so both stores pass the
    identical suite (SC-003).
- **Rationale**: User input. A single seam keeps endpoints storage-agnostic, and the shared suite
  enforces FR-019 parity.
- **Alternatives**: separate repositories per entity (more types, no benefit at this size).

## R9 — Atomic order placement and no overselling (FR-013–FR-015, SC-004)

- **Decision**: The client sends lines `{itemId, quantity, unitPriceCents}`. The store checks
  existence, price and stock and either writes everything or nothing.
  - The client's `unitPriceCents` is only the **expected** price for the FR-013 comparison.
    Stored line prices and the total are always taken from the stored item, never from the
    request. Orders whose total would exceed 2^53 − 1 cents are refused (400), so both stores
    compute it exactly.
  - **PostgreSQL**: one transaction. `SELECT … FOR UPDATE` on the items in id order (no
    deadlocks), compare, then `UPDATE items SET stock = stock - q` and `INSERT` the order.
  - **Redis**: one Lua script (`EVALSHA`). Read each item from hash `shop:items` (cjson), compare,
    then `HSET` the new stock, `SET` the order and `ZADD` it to the index. Lua runs atomically.
  - Any mismatch returns the list of problems (`removed`, `price`, `stock`) and changes nothing.
- **Rationale**: Both are atomic in a single round trip; the SC-004 test (20 concurrent orders, 5
  in stock) runs against both.
- **Alternatives**: optimistic retries with `WATCH`/`MULTI` (more code and retries under
  contention); app-level locks (break with 2–3 replicas).

## R10 — Reading the operator's settings

- **Decision**: At start, validate:
  - `SHOP_NAME`, `WALLET_ADDRESS` and `ADMIN_PASSWORD` are present;
  - exactly one of `DATABASE_URL` / `REDIS_URL` is set;
  - `REDIS_PASSWORD` is present when `REDIS_URL` is.

  Convert the CNPG `uri` (`postgresql://user:pw@host:5432/db`) into an Npgsql connection string.
  Build the StackExchange.Redis configuration from host, port and password with
  `abortConnect=false`. Errors name the missing variable and never print values.
- **Rationale**: FR-019 and the "missing configuration" edge case. Npgsql doesn't parse URIs.

## R11 — Admin account, sign-in and sessions (FR-004–FR-008)

- **Decision**:
  - **Admin record**: `{hash, initial, changedAt, failedCount, failedWindowStart, lockedUntil}`.
  - **Bootstrap** (under the init lock):
    - no record → create from `ADMIN_PASSWORD`, `initial=true`;
    - `initial=true` and the hash doesn't verify `ADMIN_PASSWORD` → re-hash (the Secret was
      regenerated, R1);
    - `initial=false` → never touch it.
  - **Sign-in**: password only; 5 failures within 15 min → `lockedUntil = now+15 min`. The check
    is atomic in the store, so it holds across replicas, and a correct password is refused while
    locked (SC-006).
  - **Session**: ASP.NET Core cookie auth, `HttpOnly`, `SameSite=Strict`, `Secure` when the
    request is HTTPS, 8 h absolute expiry.
  - **Data-Protection keys** are kept through `IShopStore` (a custom `IXmlRepository`), so every
    replica can read every cookie (FR-007).
  - **CSRF**: ASP.NET antiforgery with Angular's `XSRF-TOKEN` cookie / `X-XSRF-TOKEN` header
    convention on admin state-changing endpoints.
  - **Changed passwords** need at least 12 characters and set `initial=false`.
- **Rationale**: The simplest secure option for one admin without a user table. PBKDF2 via
  `PasswordHasher` is built in. Store-backed keys avoid an extra Secret (no operator change).
- **Alternatives**: JWT in local storage (XSS exposure, no server-side expiry); ASP.NET Identity
  with its full schema (far too much for one account); keys in a per-pod folder (sessions break
  across replicas).

## R12 — Schema set-up with several replicas starting at once (FR-022)

- **Decision**: `InitializeAsync` runs before Kestrel starts listening, so the operator's TCP
  readiness probe fails until the store is ready.
  - It retries with backoff until the database answers, logging each wait once.
  - **PostgreSQL**: `pg_advisory_lock(<constant>)`, then numbered embedded SQL scripts recorded in
    `schema_version`.
  - **Redis**: `SET shop:lock:init <id> NX PX 30000` with wait-and-retry, then a version number in
    `shop:schema`.
  - Admin bootstrap (R11) runs inside the same lock.
- **Rationale**: The probe can only see an open port, so not listening is the readiness signal.
  Locks make concurrent starts safe (edge case 1).

## R13 — Frontend and image

- **Decision**:
  - **Frontend**: Angular (current stable major at scaffold time, exact versions in
    `package-lock.json`) with standalone components, the router and `HttpClient`; plain CSS, no UI
    library. The cart lives in `localStorage`.
  - **Image**: a multi-stage Dockerfile:
    - Node LTS builds Angular into the API's `wwwroot`;
    - the .NET SDK publishes the API;
    - the runtime is `mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled`, non-root.
    - All base images are pinned by digest.
  - **Settings**: `ASPNETCORE_HTTP_PORTS=80`, workstation non-concurrent GC and
    `InvariantGlobalization`, to keep the footprint well inside the 256 Mi limit from operator
    0.2.0 (R16).
  - **Static serving**: ASP.NET serves the files and falls back to `index.html` for client routes.
- **Rationale**: One image (FR-023), smallest attack surface, and a memory footprint that fits.
- **Alternatives**: a separate nginx for the UI (two containers, an operator change); Alpine
  images (musl quirks, larger than chiseled).

## R14 — Tests and CI (FR-025, Constitution III)

- **Decision**:
  - **Tests**: xUnit v3 with Testcontainers for .NET (`Testcontainers.PostgreSql`,
    `Testcontainers.Redis`):
    - store contract suite per store (R8);
    - API tests through `WebApplicationFactory` against each store;
    - unit tests for validation, settings parsing and lockout timing (fake `TimeProvider`).
  - **Frontend tests**: `ng test` for the cart service and the order-problem mapping.
  - **CI** (`.github/workflows/ci.yaml`), mirroring shop-operator:
    - `lint` (`dotnet format --verify-no-changes`, `ng lint`) and `test` (`dotnet test`, `ng test`)
      on PRs, both required;
    - `release` on `v*.*.*` tags: build, push `ghcr.io/nikola031ue/shop:X.Y.Z` with
      `GITHUB_TOKEN`, refuse if the tag isn't SemVer.
  - **Repo settings**: branch protection, then squash-only. The package is made public (asking
    first, as in 002).
- **Rationale**: Requirement 5.2 and Constitution II/III. GitHub's Ubuntu runners have Docker for
  Testcontainers.

## R16 — Shop pod memory (operator 0.2.0)

- **Decision**: Operator 0.2.0 sets the shop container's memory to request `128Mi`, limit
  `256Mi`, replacing 32Mi/128Mi from 0.1.0. The CPU request stays at `10m`. Values stay
  constants in `internal/resources`, not chart values.
- **Rationale**: Maintainer decision: the app must not depend on fitting 128 Mi. A .NET runtime
  plus Kestrel, the Npgsql/Redis clients and static files commonly idles at 60–100 Mi and peaks
  higher under GC. A 128Mi request makes the scheduler reserve what the app really uses. The
  change ships in the same release as R1–R5, so no extra release.
- **Alternatives**: keep 128Mi and tune the GC (fragile, an OOM risk under load); make the
  values configurable per chart (more surface than needed now).

## R15 — Rolling the app out

- **Decision**: After `v0.1.0` of `shop` is published, one kube-state PR sets
  `clusters/local/shop-operator/values.yaml` → `shopImage: {repository: ghcr.io/nikola031ue/shop,
  tag: "0.1.0"}`. bring-up upgrades only `shop-operator`, the operator restarts, and every Shop
  rolls to the app. `helm-charts/charts/shop` stays an untouched placeholder (FR-026).
- **Rationale**: The image is an operator setting (002); kube-state is the only way to change
  platform state (Constitution V).
