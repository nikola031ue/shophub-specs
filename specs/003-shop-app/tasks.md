---

description: "Task list for 003-shop-app"
---

# Tasks: Shop Application (Requirements 2.1, 2.2, 2.3)

**Input**: Design documents from `/specs/003-shop-app/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Required. Constitution Principle III mandates unit tests plus envtest (operator) and
Testcontainers (app) integration tests before merge. FR-025 names both databases.

**Organization**:

- Phase 1 is operator 0.2.0 alone. It is released and deployed via kube-state **before any app
  task** (spec Clarifications).
- App phases follow. Story phases are in priority order: US1 → US2 → US3 → US4.
- Every merge, GitHub settings change and package visibility change is asked of the maintainer
  first. Commits are made at the end of each phase. Nothing is pushed unless the maintainer asks.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: User story the task belongs to (US1–US4)
- Paths: `shop-operator/…`, `helm-charts/…` and `kube-state/…` are the existing sibling repos;
  `shop/…` is the app repo ([contracts/delivery.md](contracts/delivery.md))

---

## Phase 1: Operator 0.2.0 (blocks all app work)

**Purpose**: Admin Secret, Redis auth, Redis persistence, port-80 sysctl, memory 128Mi/256Mi, and
automatic upgrade of 0.1.0 Shops ([contracts/operator-0.2.0.md](contracts/operator-0.2.0.md),
research R1–R6 and R16).

- [X] T001 [P] Builders in `shop-operator/internal/resources/`:
  - new `secrets.go`: the `<shop>-admin` and `<shop>-redis-auth` Secret builders with
    `crypto/rand` values (24 and 32 URL-safe chars), labels and names;
  - `resources.go`:
    - Deployment env `ADMIN_PASSWORD` and (light) `REDIS_PASSWORD` via `secretKeyRef`;
    - pod sysctl `net.ipv4.ip_unprivileged_port_start=0`;
    - annotations `shophub.io/admin-secret-uid` and `shophub.io/redis-secret-uid`;
    - memory request `128Mi`, limit `256Mi`;
    - Redis `kubernetesConfig.redisSecret`, `storage` (1Gi RWO, `keepAfterDelete: false`,
      PVC labels come from the Redis CR via the Redis operator; see log) and
      `redisConfig.dynamicConfig: ["appendonly yes"]`.

  Unit tests in `internal/resources/resources_test.go`, including: no secret value in any
  annotation or plain env value; values differ per call.
- [X] T002 Shop reconciler in `shop-operator/internal/controller/shop_controller.go`, plus the
  reason `RedisCredentialsLost` in `common.go`:
  - ensure the admin Secret, with `APIReader` confirmation and regeneration when it is missing;
  - ensure the Redis Secret, never regenerated while `<shop>-redis` exists (→
    `RedisCredentialsLost`);
  - one-time delete of a live Redis without `spec.storage` (0.1.0 upgrade, info log, no event);
  - `Owns(&corev1.Secret{})`.

  envtest specs in `shop_controller_test.go` cover each rule, the UID annotation changing after an
  admin Secret is recreated, and Secrets garbage-collected with the Shop (owner refs).
- [X] T003 Docs:
  - `shop-operator/README.md`: env contract table, `RedisCredentialsLost` recovery (delete and
    recreate the Shop), note on the 0.1.0 Redis recreate;
  - a "0.2.0" note in `specs/002-shop-operator/contracts/crds.md` linking to
    `specs/003-shop-app/contracts/operator-0.2.0.md`.

  Then push the branch and open the shop-operator-spec PR (maintainer asks to push), wait for green
  `lint` and `test`, **ask before merging**, tag `v0.2.0`, and verify an anonymous
  `docker pull ghcr.io/nikola031ue/shop-operator:0.2.0`.
- [X] T004 helm-charts:
  - run `shop-operator/hack/sync-chart.sh` (expected: no CRD or RBAC diff);
  - `helm-charts/charts/shop-operator/Chart.yaml` → version `0.3.0`, appVersion `0.2.0`;
  - update `NOTES.txt` if it mentions env/Secrets.

  Open the helm-charts-spec PR, CI green, **ask before merging**, and confirm chart `0.3.0` is
  published and public.
- [X] T005 kube-state upgrade (quickstart Scenarios 1–2):
  - **before merging**, create test Shops `std` (standard) and `lite` (light) in `qs` on the
    0.1.0 operator and wait for Ready;
  - `kube-state/clusters/local/shop-operator/helm.yaml` → `0.3.0`; open the kube-state-spec PR,
    CI green, **ask before merging**;
  - run `./bring-up.sh` and verify every Scenario 2 check (Secrets, NOAUTH/PONG, `appendonly
    yes`, Bound PVC, standard DB pod UID unchanged, 128Mi/256Mi, `RedisCredentialsLost` drill);
  - record the results in the log below. Keep `std` and `lite` for later scenarios.

**Checkpoint**: operator 0.2.0 runs on the cluster and 0.1.0 Shops were upgraded with no hand
change. App work may start.

---

## Phase 2: App setup (`shop` repo)

**Purpose**: Scaffold, image and CI before any feature code ([contracts/delivery.md](contracts/delivery.md),
research R7, R13, R14).

- [X] T006 Scaffold `shop/`:
  - `Shop.slnx`, `global.json` (.NET 10 SDK pinned), `Directory.Build.props` (nullable, warnings
    as errors, `InvariantGlobalization`, workstation non-concurrent GC) and
    `Directory.Packages.props` (exact versions, lock files);
  - `src/Shop.Api` (minimal API, `wwwroot` fallback to `index.html`) and
    `tests/Shop.Api.Tests` (xUnit v3, Testcontainers);
  - `src/Shop.Web`: Angular standalone app with router, HttpClient with XSRF, angular-eslint;
    build output to `src/Shop.Api/wwwroot`;
  - `.gitignore`, `.dockerignore`, `.editorconfig`.
- [X] T007 [P] `shop/Dockerfile`, multi-stage:
  - Node LTS builds Angular, the .NET SDK publishes, the runtime is
    `aspnet:10.0-noble-chiseled`, non-root;
  - every base image pinned by digest; `ASPNETCORE_HTTP_PORTS=80`; OCI labels.

  Local check: `docker build` and `docker run -e …` with local Postgres serves `index.html` on
  port 80.
- [X] T008 `shop/.github/workflows/ci.yaml`:
  - jobs `lint` (`dotnet format --verify-no-changes`, `ng lint`), `test` (`dotnet test`,
    `ng test --watch=false`) and `release` (`v*.*.*` only, SemVer check, push
    `ghcr.io/nikola031ue/shop:X.Y.Z` with `GITHUB_TOKEN`, never `latest`);
  - actions pinned by SHA.

  Then open the first shop-spec PR. **Ask before** setting branch protection (required `lint`,
  `test`; linear history; no force push), squash-only and auto-delete. Then **ask before
  merging**.

---

## Phase 3: Foundational (blocks US1–US4)

**Purpose**: Settings, domain rules, the single store abstraction with both implementations and the
shared contract suite, startup and admin authentication (research R8–R12).

- [X] T009 [P] `shop/src/Shop.Api/Settings.cs` (research R10):
  - validate `SHOP_NAME`, `WALLET_ADDRESS`, `ADMIN_PASSWORD`, exactly one of `DATABASE_URL` /
    `REDIS_URL`, and `REDIS_PASSWORD` when `REDIS_URL` is set;
  - convert the CNPG URI to an Npgsql connection string and build the StackExchange.Redis options
    (`abortConnect=false`);
  - errors name the variable, never the value.

  Unit tests in `tests/Shop.Api.Tests/Unit/SettingsTests.cs`.
- [X] T010 [P] Domain in `shop/src/Shop.Api/Domain/`, enforcing these data-model.md fields:
  - `Item.id` (UUIDv7 generated by the app), `Order.createdAt` (UTC, set at placement);
  - `Item.name` (trimmed, 1–100, case-insensitive uniqueness key `nameKey`), `Item.stock` (≥ 0),
    `Item.priceCents` (≥ 1, decimal input ≤ 2 places, mUSDT cents);
  - `Order.reference` (10-char Crockford base32 from a CSPRNG), `Order.status` (`PendingPayment`
    only), `Order.walletAddress`, `Order.totalCents`, `Order.lines` (1–50, duplicates merged);
  - `OrderLine.quantity` (≥ 1), `OrderLine.unitPriceCents`, `OrderLine.name`.

  Unit tests in `tests/Shop.Api.Tests/Unit/DomainTests.cs`.
- [X] T011 Store seam:
  - `shop/src/Shop.Api/Storage/IShopStore.cs`: items, place order (problems: removed, price,
    stock), list orders, admin record, Data-Protection keys, `InitializeAsync`;
  - abstract `tests/Shop.Api.Tests/Integration/ShopStoreContract.cs` with every behavioural test,
    including:
    - the data-model.md `Admin.failedCount` / `Admin.failedWindowStart` / `Admin.lockedUntil`
      atomic lockout update (window of 15 min starts at the first failure);
    - data-model.md `Admin.changedAt` set on password change;
    - concurrent `InitializeAsync` from 3 tasks;
    - **SC-004** (20 concurrent orders, 5 in stock → exactly 5 succeed);
    - order snapshot after item rename and delete.
- [X] T012 [P] `shop/src/Shop.Api/Storage/Postgres/` (`PostgresShopStore`, embedded numbered SQL
  scripts, `schema_version`, `pg_advisory_lock`):
  - order placement in one transaction with `FOR UPDATE` in id order;
  - `ILIKE` search with escaping;
  - pin the Testcontainers PostgreSQL image to the same major version as the CNPG operator's
    default image (read it from the installed CNPG release; note it in the log).

  Implements the PostgreSQL layout in data-model.md. `Integration/PostgresStoreTests.cs` subclasses
  the contract with a Testcontainers PostgreSQL container (same major as the CNPG default).
- [X] T013 [P] `shop/src/Shop.Api/Storage/Redis/` (`RedisShopStore` and Lua scripts as embedded
  resources):
  - create, rename, delete and place-order scripts keep `shop:items`, `shop:item-names`,
    `shop:order:*` and `shop:orders` consistent;
  - `NX PX` init lock; `shop:schema`.

  Implements the Redis layout in data-model.md. `Integration/RedisStoreTests.cs` subclasses the
  contract with a Testcontainers Redis 7.2 container started with `--requirepass`.
- [X] T014 Startup and auth (research R11, R12):
  - `Program.cs`:
    - pick the store from Settings and run `InitializeAsync` with retry/backoff (capped at
      10 s between attempts, so the app is usable within 1 min of the database answering)
      **before** `app.Run()`;
    - problem details, generic 500 with a correlation id, log redaction of connection strings;
    - `/healthz`.
  - `Auth/AdminAuth.cs`: bootstrap rules for the data-model.md `Admin.passwordHash` /
    `Admin.initial` fields, `PasswordHasher`, password change setting data-model.md
    `Admin.changedAt`, lockout via the store using data-model.md `Admin.failedCount` /
    `Admin.failedWindowStart` / `Admin.lockedUntil` (5 per 15 min, `TimeProvider`),
    cookie auth (`HttpOnly`, `SameSite=Strict`, 8 h absolute), antiforgery.
  - `Auth/StoreXmlRepository.cs` (Data-Protection keys in the store).

  Unit tests for bootstrap and lockout timing in `Unit/AdminAuthTests.cs`.

**Checkpoint**: both stores pass the identical contract suite; the app starts on either database.

---

## Phase 4: User Story 1 — Admin manages items (P1) 🎯 MVP

**Goal**: The admin signs in with the generated password, changes it and manages items (FR-001–FR-008).

**Independent Test**: The API tests below on both stores; quickstart Scenario 5.

- [X] T015 [US1] Admin endpoints in `shop/src/Shop.Api/Endpoints/Admin.cs` per
  [contracts/http-api.md](contracts/http-api.md):
  - `login` (204/401/429 with `Retry-After`), `logout`, `me`, `password` (≥ 12 chars, sets
    `initial=false`);
  - items list, create, update and delete, with 409 on duplicate name and per-field 400s.

  `Integration/ApiTests.cs`, run once per store via `WebApplicationFactory`, covers US1 scenarios
  1–6 and:
  - 401 without a session and 400 without the XSRF header;
  - the sixth login refused while correct (SC-006);
  - a session cookie from one app instance accepted by a second instance on the same store
    (FR-007);
  - a changed password surviving a restart;
  - **FR-024 / SC-007**: all app log output captured during the API tests and a failed start
    with a bad `DATABASE_URL` contains none of `ADMIN_PASSWORD`, `REDIS_PASSWORD`, the DB
    password, the admin password hash, or any password the tests sent.
- [X] T016 [P] [US1] Angular admin in `shop/src/Shop.Web/src/app/admin/`:
  - login page, auth guard, prompt to change the initial password;
  - items list with search, create/edit form with field errors, delete with confirm;
  - change-password page, sign-out.

---

## Phase 5: User Story 2 — Customer finds items and places an order (P1)

**Goal**: Search, cart and an atomic `PendingPayment` order (FR-009–FR-017).

**Independent Test**: The API tests below on both stores; quickstart Scenario 6.

- [X] T017 [US2] Public endpoints in `shop/src/Shop.Api/Endpoints/Public.cs`:
  - `GET /api/shop`, `GET /api/items?q=&page=`;
  - `POST /api/orders` → 201 / 409 problems (nothing stored) / 400 / 503.

  `ApiTests.cs` covers US2 scenarios 1, 4, 5 and 6 on both stores: stock lowered by exactly the
  ordered quantities; `walletAddress` on the order equals `WALLET_ADDRESS`; no auth needed.
- [X] T018 [P] [US2] Angular storefront in `shop/src/Shop.Web/src/app/store/`:
  - shop name header, item list with search, unavailable state;
  - cart service on `localStorage["shop.cart"]` with quantities capped at stock;
  - cart page, checkout that maps 409 problems to "removed / price changed / only N left" with a
    confirm-and-retry;
  - order confirmation showing reference and total.

  `ng test` specs for the cart service and the problem mapping.

---

## Phase 6: User Story 3 — Admin lists orders (P2)

**Goal**: Orders newest first, 20 per page, with snapshot lines (FR-018).

**Independent Test**: The API tests below; quickstart Scenario 7.

- [X] T019 [US3] `GET /api/admin/orders?page=` in `shop/src/Shop.Api/Endpoints/Admin.cs`. `ApiTests.cs`
  on both stores covers:
  - 3 orders newest first;
  - 45 orders → pages of 20, 20 and 5 with each order exactly once;
  - snapshot name and price after the item is renamed or deleted;
  - 401 without a session.
- [X] T020 [P] [US3] Angular orders page in `shop/src/Shop.Web/src/app/admin/orders/`: table with
  reference, time, status, total and expandable lines; pager.

---

## Phase 7: User Story 4 — Release and run on both modes (P2)

**Goal**: A released image runs on standard and light shops via the operator (FR-019–FR-026,
SC-003, SC-007–SC-009).

**Independent Test**: quickstart Scenarios 3–9 on the local cluster.

- [X] T021 [US4] `shop/README.md`:
  - local run against Testcontainers or Docker Postgres/Redis;
  - the settings table (from contracts/operator-0.2.0.md);
  - tests, release steps, the admin-password flow and the lockout;
  - "do not paste the admin password anywhere".
- [X] T022 [US4] Release:
  - push the branch when the maintainer asks, open the shop-spec PR, CI green, **ask before
    merging**;
  - tag `v0.1.0` and confirm the `release` job pushed `ghcr.io/nikola031ue/shop:0.1.0` within
    15 min (SC-008);
  - **ask before** making the GHCR package public; verify an anonymous `docker pull`.
- [X] T026 Operator 0.2.1: self-healing Redis password (planned 2026-11-24; ID added after
  T025, runs here). Can start any time after Phase 1 and **must finish before T024**:
  - in `shop-operator/internal/controller/shop_controller.go` `prepareRedis`: when
    `<shop>-redis-auth` is missing and `<shop>-redis` exists, generate a new password Secret
    instead of reporting `RedisCredentialsLost`. Restart the Redis so it starts with the new
    password, keeping the PVC and its AOF data.
    - Restart path (maintainer, 2026-11-24): **prefer** switching the Redis CR's
      `kubernetesConfig.redisSecret.name` to a **new Secret name** (for example a generation
      suffix, `<shop>-redis-auth-<n>`) over adding pod-delete RBAC. First verify, in envtest
      where possible and live in `qs`, that the OT redis-operator rolls the Redis StatefulSet
      on that change and that the data on the PVC survives. Only if it does **not** roll out,
      fall back to deleting pod `<shop>-redis-0`, with RBAC scoped as narrowly as possible
      (pods `delete` only, no wider verbs). Record which path was taken and why. Never delete
      the Redis CR or the PVC.
    - With a new Secret name, the env `REDIS_PASSWORD` `secretKeyRef` and the old Secret's
      cleanup must follow it.
    - The shop pods roll on their own through the `shophub.io/redis-secret-uid` annotation.
  - Retire or redefine `ReasonRedisCredentialsLost` (Ready=False only while the Redis restarts).
    Log the Secret name only.
  - envtest: Secret recreated, Redis CR and PVC not deleted, annotation changes. Live drill in
    `qs`: write a key, delete `lite-redis-auth`, check the key survives and `lite` is Ready with
    the new password.
  - Docs: README recovery section, [contracts/operator-0.2.0.md](contracts/operator-0.2.0.md)
    rule 2 and status table, research R2, quickstart Scenario 2 step 4.
  - Memory 128Mi/256Mi is already in 0.2.0 (verified in code and on `std`/`lite`); nothing to add.
  - Release: shop-operator PR, **ask before merging**, tag `v0.2.1`; chart `shop-operator`
    `0.3.1`; kube-state `helm.yaml` → `0.3.1` (may share the T023 PR). Each merge asked first.
- [X] T023 [US4] kube-state rollout:
  - `kube-state/clusters/local/shop-operator/values.yaml` adds
    `shopImage: {repository: ghcr.io/nikola031ue/shop, tag: "0.1.0"}`;
  - open the kube-state-spec PR, CI green, **ask before merging**;
  - run `./bring-up.sh` (only `shop-operator` upgraded); `std` and `lite` roll to the app and
    reach Ready.
- [X] T024 [US4] Quickstart Scenarios 3–9 on Docker Desktop:
  - seed 1,000 items through the admin API on each shop and time a search (SC-005, quickstart
    Scenario 6 step 4);
  - record timings (SC-001, SC-002, SC-005, SC-008) and results in a Validation log in
    `specs/003-shop-app/quickstart.md`;
  - delete namespace `qs` afterwards.

---

## Phase 8: Polish

- [ ] T025 Self-review:
  - check the merged diffs in shop-operator, helm-charts, kube-state and shop against the plan's
    Constitution Check and Complexity Tracking;
  - `git grep` all repo histories for the admin/Redis passwords, DB URIs and other secret-looking
    values (SC-007);
  - record the outcome and any deviations in the log below; set `spec.md` Status to Implemented.

---

## Dependencies & Execution Order

### Phase Dependencies

- **Phase 1 (operator 0.2.0)**: T001 → T002 → T003 → T004 → T005. Nothing in Phase 2+ starts
  before T005 is done (spec Clarifications).
- **Phase 2 (setup)**: T006 → T007 ∥ T008. T008 holds the first merge of `shop`, so branch
  protection comes before it (Principle II).
- **Phase 3 (foundational)**: T009 ∥ T010 → T011 → T012 ∥ T013 → T014.
- **Phases 4–6 (US1–US3)**: each needs Phase 3. US3 needs orders from US2 to be meaningful but its
  tests seed orders through the store, so it can run after Phase 3.
- **Phase 7 (US4)**: needs US1–US3 merged-ready. T021 → T022 → T023 → T024. T026 (operator
  0.2.1) depends only on Phase 1 and must be deployed before T024.
- **Phase 8**: after T024.

### Within Each Story

- The API task (with its tests) before the UI task's manual check. UI tasks are [P] with their API
  task because they live in `src/Shop.Web`.

### Parallel Opportunities

- T001 runs alongside the T002 test scaffolding (separate packages).
- T007 ∥ T008; T009 ∥ T010; T012 ∥ T013.
- T016 ∥ T015, T018 ∥ T017, T020 ∥ T019.

---

## Parallel Example: Foundational

```bash
Task: "T012 PostgresShopStore + PostgresStoreTests in shop/src/Shop.Api/Storage/Postgres/"
Task: "T013 RedisShopStore + Lua + RedisStoreTests in shop/src/Shop.Api/Storage/Redis/"
```

---

## Implementation Strategy

### Operator first

Phase 1 is a full release cycle of its own: operator PR → `v0.2.0` → chart `0.3.0` → kube-state
PR → live upgrade check. Stop and validate before any app task.

### MVP (User Story 1)

Phases 2–4 give a shop whose admin can manage items on both databases, proven by the shared suite.
It isn't released alone: `v0.1.0` (T022) also contains US2 and US3, so there is one rollout.

### Incremental Delivery

1. shop-operator PR → `v0.2.0`; helm-charts PR → `0.3.0`; kube-state PR → live upgrade
2. shop scaffold + CI PR (first merge, protection set)
3. shop feature PR (US1–US3) → `v0.1.0`
4. kube-state rollout PR → quickstart validation

---

## Implementation log

### 2026-11-23 — Recorded deviation (carried from spec Clarifications)

- shophub-specs PR #1 (spec 002) was merged with a merge commit, `79cc773`, contrary to
  Constitution II / Requirement 5.1 linear history. The maintainer accepted it as a one-off; `main`
  is not rewritten. Merge commits are disabled in shophub-specs (squash and rebase only).

### 2026-11-24 — Phase 1: operator 0.2.0 (T001–T005)

- **T001/T002** (shop-operator-spec#4, squash `e884b21`):
  - new `internal/resources/secrets.go`;
  - `resources.go`: env refs, sysctl, UID annotations, memory 128Mi/256Mi, Redis `redisSecret`,
    storage and AOF;
  - `shop_controller.go`: `ensurePasswordSecret` and `prepareRedis` (0.1.0 recreate,
    `RedisCredentialsLost`), `Owns(Secret)`, `APIReader`.
  - Tests: new unit tests and 3 envtest specs (`shop_secrets_test.go`). 55/55 envtest specs and
    all unit packages pass; `make lint` reports 0 issues; CI lint and test green. Generated RBAC
    is unchanged.
- **Deviation from T001 as written**: `storage.volumeClaimTemplate.metadata.labels` can't be set.
  The OT CRD gives `metadata` no properties, so the API server prunes it (the T002 envtest caught
  this). Removed from the builder. On the cluster, the Redis operator copies the Redis CR's labels
  onto the PVC (`managed-by`, `shophub.io/owner`, `shophub.io/shop`), so analyze finding C1 is
  met anyway.
- **T003**:
  - README: env contract, generated passwords, `RedisCredentialsLost` recovery, 0.1.0 upgrade;
  - "Operator 0.2.0" note in `002/contracts/crds.md`;
  - tag `v0.2.0`, release job green; anonymous pull OK, digest
    `sha256:254e1485b64ba187c6b2d5539a28d4b439ccf0b1f18d7194979be69eab79f22a`.
- **T004** (helm-charts-spec#6, `030615b`): `sync-chart.sh` changed only the generated-file
  headers (no CRD or RBAC diff). Chart `shop-operator` `0.3.0` / appVersion `0.2.0`; publish and
  verify-public green; anonymous `helm show chart` OK.
- **T005** (kube-state-spec#5, `3909bfa`):
  - Before the merge, `std` (standard) and `lite` (light) were Ready on operator 0.1.0 in `qs` in
    31 s (shared Wallet `w`, DiscordChannel `qs-alerts`).
  - `./bring-up.sh`: only `shop-operator` was upgraded (37 s). Both shops were Ready again on
    0.2.0 about 46 s later, with no hand change.
  - Operator log: `Created generated password Secret` for `std-admin`, `lite-admin` and
    `lite-redis-auth`; `Recreating Redis without storage (0.1.0 upgrade)` for `lite-redis`, once.
  - Secrets are owned by their Shop and carry the managed-by, owner and shop labels.
  - Deployments: `ADMIN_PASSWORD` and `REDIS_PASSWORD` via `secretKeyRef`; `REDIS_URL` without
    credentials; memory 128Mi/256Mi; sysctl present; UID annotations present.
  - `std` database pod UID unchanged (`97b372bd…`).
  - Redis:
    - PVC `lite-redis-lite-redis-0` Bound, 1Gi, with the shop labels;
    - `CONFIG GET appendonly` → `yes`;
    - `redis-cli ping` without credentials → `NOAUTH Authentication required`, in the pod and via
      the Service;
    - with the password → `PONG`;
    - the container exports `REDISCLI_AUTH`, so a plain `redis-cli` in the pod looks
      unauthenticated; test with `env -u REDISCLI_AUTH`.
  - Extra check (hand change in `qs`): a key written before deleting the `lite-redis-0` pod was
    still there afterwards.
  - Drill: deleting `lite-redis-auth` → `RedisCredentialsLost` with the recovery hint, and no new
    Secret. Recovery (delete and recreate `lite`) removed the old Redis and PVC and made `lite`
    Ready in 36 s.
  - Leak scan: the three passwords appear 0 times in the operator log (1 h), the Shop/Redis YAML
    and the `qs` events.
  - `std` and `lite` are kept in `qs` for Phase 7.

### 2026-11-24 — Phase 2: app setup (T006–T008)

- Before starting: memory request 128Mi / limit 256Mi confirmed in operator 0.2.0
  (`internal/resources/resources.go`) and on the live `std`/`lite` Deployments; nothing for 0.2.1.
  The self-healing Redis password is planned as T026 (operator 0.2.1, before T024).
- **T006** (shop-spec#1, squash `7e5ee76`):
  - .NET SDK `10.0.401` in `global.json`, which also opts into Microsoft.Testing.Platform. The
    .NET 10 SDK refuses VSTest with xUnit v3, so `Microsoft.NET.Test.Sdk` and
    `xunit.runner.visualstudio` are not used; tests run with `dotnet test --solution Shop.slnx`.
  - Central package versions (exact), lock files, locked restore when `CI=true`.
  - `Shop.Api`: static files with `index.html` fallback, `/healthz`; one smoke test.
  - `Shop.Web`: Angular 22.3.0 (exact versions, `save-exact`), router, `HttpClient` with
    `XSRF-TOKEN`/`X-XSRF-TOKEN`, angular-eslint 22.8.0, build output to `Shop.Api/wwwroot`
    (gitignored). `npm audit --omit=dev`: 0; 7 moderate in dev-only tooling (`minimatch`).
- **T007**: Dockerfile with `node:24-bookworm-slim`, `sdk:10.0.401-noble` and
  `aspnet:10.0-noble-chiseled`, all by digest; runs as UID 1654; `ASPNETCORE_HTTP_PORTS=80`;
  OCI labels. Local run: `/` and `/admin/items` return `index.html`, `/healthz` 200, about 20 MiB
  RSS, image 182 MB. No database check yet: the app has no store code until Phase 3.
- **T008**: `ci.yaml` with `lint`, `test` and `release` (SemVer, no overwrite, no `latest`),
  actions pinned by SHA (adds `setup-dotnet` v6.1.0 and `setup-node` v7.1.0). All four checks pass
  locally and in CI (PR and `main`).
- Repo settings (approved): `main` protected like shop-operator-spec (PR required, required
  strict checks `lint` and `test`, linear history, admins included, no force pushes or
  deletions); squash-only; head branches auto-deleted. PR #1 squash-merged after approval.

### 2026-11-24 — Phase 3: foundational (T009–T014)

- shop-spec#2 (branch `003-foundation`), squash-merged as `7273041` after the maintainer's
  approval, once the price fix below was in (CI 109 tests green on `2128157`).
- **Order prices** (maintainer review before merge, fix `2128157`): both stores used to copy the
  request's `unitPriceCents` into the stored line and total. That was only safe because a
  mismatch rejects the order.
  - Now the request value is renamed `OrderRequestLine.ExpectedUnitPriceCents` and is used only
    for the FR-013 comparison. Lines and the total use the stored item price (PostgreSQL row;
    Lua `item.priceCents`, total summed in Lua and written with `%.0f`).
  - The client value is **not ignored entirely**: FR-013 requires detecting a changed price, so
    it remains the expectation that is compared.
  - Orders whose total would exceed 2^53 − 1 cents are refused in `OrderRules.Normalize`
    (`Int128` sum), so both stores stay exact.
  - New contract tests `ClientCannotChooseThePrice` (lower or higher client prices never produce
    an order; stored line price and total equal the store's) and `LargeTotalsStayExact`.
  - Docs: data-model.md `OrderLine.unitPriceCents`, research R9, contracts/http-api.md.
- **T009** `Settings.cs`: env validation per R10; the CNPG URI goes to Npgsql with
  `GssEncryptionMode=Disable`. Running the image against a real Postgres showed Npgsql trying to
  load `libgssapi_krb5` on the chiseled runtime; CNPG doesn't use GSSAPI. Errors name variables
  only (unit tests assert no value leaks).
- **T010** Domain: prices capped at 999,999,999.99 (`Money.MaxPriceCents`), so every number the
  Redis Lua scripts handle stays exact (cjson keeps 14 significant digits); the order total is
  computed by the app and spliced into the stored JSON as text. Different prices for the same
  item in one request are a 400.
- **T011** `IShopStore` and `ShopStoreContract` (23 tests): it takes the caller's `now` for item
  and order times, stored at millisecond precision in both stores (`StoreTime`), so results
  compare equal. It includes SC-004, concurrent init from 3 replicas, concurrent lockout
  updates, snapshot after rename/delete, wildcard escaping and paging.
- **T012** PostgreSQL: CNPG default image on the cluster is
  `ghcr.io/cloudnative-pg/postgresql:18.7` (CNPG 1.30.2), so tests use `postgres:18.7`. Sorting
  uses `name_key COLLATE "C"` to match the Redis store's ordinal sort.
- **T013** Redis: tests use `redis:7.2.16`, the version the OT operator runs
  (`quay.io/opstree/redis:v7.2.16`), with `--requirepass`; an extra test checks that an
  unauthenticated client is refused.
- **T014** Startup and auth:
  - Store init is an `IHostedService`, so it completes before Kestrel listens (hosted services
    start before the server). `WebApplicationFactory` stops `Program` right after `Build()`, so
    code placed before `app.Run()` would never run in tests.
  - Backoff from 0.5 s, capped at 10 s; each distinct wait reason is logged once, redacted.
  - Errors: own middleware instead of `UseExceptionHandler`, whose built-in log line would print
    the raw exception; 503 when the database doesn't answer; otherwise 500 with `correlationId`.
  - Auth: bootstrap rules, `PasswordHasher`, lockout through the store, cookie session, XSRF
    filter (`RequireXsrf`) and `IssueXsrfCookie` for the T015 endpoints, Data-Protection keys
    in the store. ASP.NET logs "No XML encryptor configured" at start: keys are stored
    unencrypted in the shop's own database, as planned in R11 (anyone who can read that
    database can already read everything else).
- Tests: xUnit v3 on Microsoft.Testing.Platform, 105 total (unit tests for settings, domain,
  auth and backoff; contract × 2; app start-up on each database; invalid configuration).
- Image checks against real containers (throwaway, removed afterwards):
  - both modes start, initialise and serve `/` at about 28 MiB;
  - Postgres stopped: the port stays closed, and the app is healthy 3 s after the database
    starts;
  - Postgres stopped while running: `/healthz` returns 503 problem details;
  - generated passwords: 0 hits in the logs.
  - This completes the database part of the T007 local check.

### 2026-11-24 — Phase 4: User Story 1, admin manages items (T015–T016)

- shop-spec#3 (branch `003-us1-admin`), squash-merged as `e17a34f` after approval, with green CI
  on the final commit `759de2b` (135 tests).
- **T015** `Endpoints/Admin.cs` per contracts/http-api.md:
  - login 204/401/429 with `Retry-After`; logout; `me`, which also re-issues the XSRF cookie so
    a reloaded UI can make changes; password change;
  - items list, create, update and delete, with per-field 400s and 409 (`errors.name`) on a
    duplicate name.
  - Authorization runs before the XSRF filter: no session gives 401, a session without the
    header gives 400.
- `Integration/ApiTests.cs`, 9 tests per store, covers everything T015 lists. Notes:
  - The test Postgres now gets a unique password, so the log-leak check can search for it.
  - The failed start uses a wrong database password with an 8 s host startup timeout; logs are
    captured at Debug level.
  - The lockout is also checked from a second app instance.
- **T016** Angular admin under `src/app/admin/`:
  - login page, `adminGuard` (redirects with `returnUrl`), banner prompting to replace the
    generated password;
  - items page with search, pager, create/edit form showing field errors, delete with
    `confirm`;
  - change-password page (client checks length and match first), sign-out.
  - Plain CSS. `shared/api.ts` has the cents formatter and the problem/field-error mapping,
    with unit tests.
  - Editing happens inline on the items page; the contract has no single-item GET, so no edit
    route was added.
- Real image against Postgres 18.7, over HTTP with a cookie jar like a browser:
  - `/admin/items` serves the app; login 204;
  - cookies: `shop.session` and `shop.af` HttpOnly, `XSRF-TOKEN` readable;
  - create without XSRF 400; create 201; invalid input 400 naming all three fields;
  - logout, then `me` 401;
  - 0 secret hits in the logs; about 38 MiB.
- Not done yet: clicking through the UI in a real browser (planned for the T024 quickstart run).
  The storefront parts of US1 scenarios 2–4 depend on the public endpoints (T017).
- ~~Known limit (FR-007)~~ **Resolved in PR #3** (maintainer request, commit `759de2b`):
  - **Session version** on the admin record: `AdminRecord.SessionVersion`, PostgreSQL migration
    `002_session_version.sql`, Redis field `sessionVersion`. It starts at 1.
  - It is incremented atomically on sign-out (`BumpSessionVersionAsync`) and on password change
    (in the same statement or Lua script that writes the hash). The bootstrap re-hash (generated
    Secret replaced) increments it too.
  - The session cookie carries the version as a claim. `OnValidatePrincipal` refuses older
    versions on `/api/admin` requests (one store read per admin request; storefront and static
    requests don't read the store).
  - A password change signs the changing browser in again with the new version and a fresh XSRF
    token.
  - Sign-out ends **every** admin session, not just this browser's. That's acceptable with one
    admin account.
  - Tests on both stores:
    - contract: starts at 1, only goes up, kept by lockout and bootstrap writes, 30 concurrent
      bumps all counted;
    - API: a stolen cookie is refused after sign-out and after a password change, on the same
      and on a second app instance; signing in again works; the changing browser keeps working.
  - data-model.md: `Admin.sessionVersion` added (entity table and PostgreSQL layout).
  - Also fixed: a timing-dependent failure of the Redis log-leak test. With a wrong password the
    first init attempt could outlast the test's 8 s startup timeout, so the cancellation skipped
    the "waiting" log line. The initializer now logs the reason of a failed attempt even while
    the start is being cancelled.

### 2026-11-24 — Phase 5: User Story 2, customer places an order (T017–T018)

- shop-spec#4 (branch `003-us2-storefront`), squash-merged as `fe23368` after approval, with
  green CI on the final commit `922d4d1` (148 tests).
- **Added before merge** (maintainer request, commit `922d4d1`):
  - **Rate limit** on `POST /api/orders` with built-in ASP.NET Core rate limiting.
    - Fixed window per TCP peer address, 10 requests per minute by default
      (`ORDER_RATE_LIMIT_PER_MINUTE` overrides); queue 0.
    - 429 problem details with `Retry-After`; only this endpoint is limited.
    - The limit is in memory per replica (N replicas → up to N × 10). Forwarded headers are not
      trusted. Behind a Kubernetes Service the peer may be a node or proxy address rather than
      the real client, so the limit can act per node; revisit when an ingress is added
      (spec 004/006).
  - **Max quantity per order line**: `OrderRules.MaxQuantityPerLine = 1000`, checked after
    duplicate lines are merged; the cart caps the same way. The largest possible total
    (50 × 1,000 × 999,999,999.99) is now far below 2^53, and the total check stays as a guard.
  - Tests:
    - per store: 4th order from one IP within the window → 429 with `Retry-After` and nothing
      stored, while another IP and `GET /api/items` still work (the test host sets the client IP
      from a test-only header); 1,001 pieces, also split over duplicate lines, → 400;
    - unit and `ng` tests for the cap.
  - Docs: contracts/http-api.md "Order limits", data-model.md `OrderLine.quantity`.
- **Dependency for spec 005 (recorded at the maintainer's request)**: *release stock of unpaid
  orders*. Stock is decremented at order time (Q2=A) and never returned in 003; spec 005 must
  expire unpaid `PendingPayment` orders and give their stock back. It is also listed under
  spec.md "Dependencies". The rate limit and line cap only limit the exposure until then.
- **T017** `Endpoints/Public.cs`:
  - `GET /api/shop`; `GET /api/items` (same handler as the admin list);
  - `POST /api/orders`: 201, 409 problems with `null` fields omitted, 400, 503 via the error
    middleware. Up to 5 attempts on a reference collision.
  - Prices come from the store (PR #2 fix).
- `ApiTests.cs`: 5 more tests per store (16 per store in total), covering US2 scenarios 1, 4, 5
  and 6, plus request validation. Scenario 6 uses two app instances on one store.
- **T018** `src/app/store/`:
  - `ShopApi` (shop name signal), `Cart` (localStorage `shop.cart`, capped at stock, malformed
    entries dropped), `correctCart`/`orderProblems` (409 → messages plus the corrected cart);
  - `Storefront`, `CartPage` (confirm and retry), `OrderPlaced` (`/orders/:reference`, total via
    navigation state);
  - header with the shop name and cart count.
  - `ng test`: 15 tests (cart, problem mapping, header).
- Real image on Redis 7.2.16 with a password:
  - shop name and search;
  - order 201, stored with the shop's wallet;
  - a stale price → 409 with the current price;
  - last piece 201 then 409;
  - client routes 200;
  - 0 secret hits in the logs; about 39 MiB.
- 145 .NET tests pass locally with `CI=true`.

### 2026-11-24 — Phase 6: User Story 3, admin lists orders (T019–T020)

- shop-spec#5 (branch `003-us3-orders`), CI `lint` and `test` green (154 tests). Squash-merged
  as `23de8b5` after the maintainer's approval.
- **T019** `GET /api/admin/orders?page=`: 20 per page, newest first (ties broken by reference),
  lines exactly as stored; 400 for `page < 1`; 401 without a session.
- `ApiTests.cs`: 3 more tests per store (20 per store in total) for US3 scenarios 1–3. The
  anonymous-refusal test now includes the orders list.
  - Orders for scenarios 1–2 are seeded through the store with distinct times, because the
    public endpoint is rate-limited.
  - Scenario 3 places its order through `POST /api/orders`, then renames, reprices and deletes
    the items.
- **T020** `src/app/admin/orders/`:
  - orders page linked from the admin nav: reference, local time, status label, total,
    expandable lines (`aria-expanded`) with line totals and the payment address; pager.
  - One component test (list, expand, next page). 17 `ng` tests in total.
- Real image on Postgres 18.7:
  - admin order list with page size 20; 401 without a session; `/admin/orders` serves the app;
  - quantity 1,001 → 400;
  - the 10th order request in a minute → 429 with `Retry-After: 60` (the earlier 400 counted
    towards the limit, as documented);
  - 0 secret hits in the logs.

### 2026-11-24 — Phase 7: README and release 0.1.0 (T021–T022)

- Browser click-through by the maintainer before the release: image built from `main`
  (`23de8b5`) on Postgres 18.7 in Docker, admin password read from a local file only. Approved;
  the review containers and the password file were removed afterwards.
- **T021** `shop/README.md`: settings table, admin-password flow with the "do not paste" warning,
  sessions and lockout, order rules, local run, image run, tests, layout, releases.
  shop-spec#6, CI green (154 tests), squash-merged as `ff4c9f3` after approval.
- **T022** tag `v0.1.0` on `ff4c9f3`, pushed 12:11:19 UTC. Release run 45391109635: `lint`,
  `test`, then `release` finished 12:14:01 UTC, **2 min 42 s** after the tag (SC-008: ≤ 15 min).
  - `ghcr.io/nikola031ue/shop:0.1.0`, digest
    `sha256:ad30c64d0f85ce0cfd94ce5291cb42bff9a0690f2645595865d5f175cd025bf9`, with provenance.
  - No `latest` tag (anonymous manifest request → 404).
  - Anonymous pull works after `docker logout ghcr.io` (registry token without credentials →
    manifest 200), so the package was already public; no visibility change was made.

### 2026-11-24 — T026: operator 0.2.1, self-healing Redis password

- **Restart path: new Secret name, no pod-delete RBAC.** Checked live first with a throwaway
  Redis CR (`probe-redis`) in `qs` on redis-operator v0.27.0. Switching
  `redisSecret.name` made the operator update the StatefulSet `secretKeyRef` and replace the pod
  in about 4 s. A key written before was still there, and the old password was refused. The
  probe was deleted afterwards.
- shop-operator `prepareRedis`:
  - the current Secret is the one the live Redis references;
  - if it is missing, a new password goes under the next name (`<shop>-redis-auth-<n>`) and the
    Redis is pointed at it;
  - the Deployment env and the UID annotation follow it;
  - the Redis CR and PVC are never deleted.
- Light-shop readiness also waits for the Redis StatefulSet rollout. `RedisCredentialsLost` is
  retired.
- Tests:
  - envtest: two successive regenerations, plus readiness during a rollout. 57 specs pass,
    82.9 % coverage.
  - Unit tests for name generations. `make lint` clean.
- Releases:
  - shop-operator-spec#5 squash-merged (`be0d4f2`), tag `v0.2.1`, image published;
  - helm-charts-spec#7 squash-merged: chart `shop-operator` 0.3.1, CRDs and RBAC unchanged,
    published and verified public;
  - kube-state-spec#6 squash-merged (`6c8f73a`): `helm.yaml` → 0.3.1. GitHub showed the `layout`
    job as stuck `in_progress` after all its steps succeeded; the PR was `CLEAN`, so it was
    merged without a re-run.
  - Each merge was approved by the maintainer.
- `./bring-up.sh`: only `shop-operator` was upgraded; `std` and `lite` stayed Ready.
- Live drill on `lite` in `qs`:
  - wrote `drill:key`, then deleted `lite-redis-auth`;
  - within about 4 s: `lite-redis-auth-2` owned by `lite`, Redis pod replaced, Redis CR and PVC
    UIDs unchanged, StatefulSet revisions equal, key still there;
  - the `lite` Deployment rolled to `lite-redis-auth-2` (annotation = new Secret UID); `lite`
    Ready;
  - the operator log names both Secrets; 0 password hits.
- Docs updated: operator README (recovery section, env table), contracts/operator-0.2.0.md
  rule 2 and the status table, research R2, quickstart Scenario 1 list and Scenario 2 step 4.

### 2026-11-24 — T023: kube-state rollout of shop 0.1.0

- kube-state-spec#7 (`shopImage: {repository: ghcr.io/nikola031ue/shop, tag: "0.1.0"}`), all 7
  checks green, squash-merged (`79e0688`) after approval.
- `./bring-up.sh`: only `shop-operator` was upgraded (values change). `std` and `lite` rolled from
  `nginx:1.27.3` to `ghcr.io/nikola031ue/shop:0.1.0` and were both Ready **27 s** after
  bring-up, with 2/2 pods each.
- In-cluster: `/healthz` ok and `/api/shop` returns the shop name on both (`QS Standard`,
  `QS Light`).

### 2026-11-24 — T024: quickstart Scenarios 3–9

- All pass on `std` and `lite`. Details and timings are in the
  [quickstart Validation log](quickstart.md#validation-log). In short:
  - SC-008: release in 2 min 42 s;
  - SC-002: order placed in about 1 s (scripted);
  - SC-005: search over 1,003 items ≤ 22 ms;
  - SC-009: data kept after a Redis pod loss and app restarts, and one session works on all 3
    replicas;
  - SC-007: 0 secret matches.
- Test-script mistake, not an app bug: the first SC-005 search used `widget 012`, but the seeded
  names have 3 digits, so it correctly matched 1 item. Re-run with `widget 01` (10 matches).
- Namespace `qs` deleted afterwards; no PVs left.
