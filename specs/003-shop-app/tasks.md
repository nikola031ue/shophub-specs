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

- [ ] T006 Scaffold `shop/`:
  - `Shop.slnx`, `global.json` (.NET 10 SDK pinned), `Directory.Build.props` (nullable, warnings
    as errors, `InvariantGlobalization`, workstation non-concurrent GC) and
    `Directory.Packages.props` (exact versions, lock files);
  - `src/Shop.Api` (minimal API, `wwwroot` fallback to `index.html`) and
    `tests/Shop.Api.Tests` (xUnit v3, Testcontainers);
  - `src/Shop.Web`: Angular standalone app with router, HttpClient with XSRF, angular-eslint;
    build output to `src/Shop.Api/wwwroot`;
  - `.gitignore`, `.dockerignore`, `.editorconfig`.
- [ ] T007 [P] `shop/Dockerfile`, multi-stage:
  - Node LTS builds Angular, the .NET SDK publishes, the runtime is
    `aspnet:10.0-noble-chiseled`, non-root;
  - every base image pinned by digest; `ASPNETCORE_HTTP_PORTS=80`; OCI labels.

  Local check: `docker build` and `docker run -e …` with local Postgres serves `index.html` on
  port 80.
- [ ] T008 `shop/.github/workflows/ci.yaml`:
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

- [ ] T009 [P] `shop/src/Shop.Api/Settings.cs` (research R10):
  - validate `SHOP_NAME`, `WALLET_ADDRESS`, `ADMIN_PASSWORD`, exactly one of `DATABASE_URL` /
    `REDIS_URL`, and `REDIS_PASSWORD` when `REDIS_URL` is set;
  - convert the CNPG URI to an Npgsql connection string and build the StackExchange.Redis options
    (`abortConnect=false`);
  - errors name the variable, never the value.

  Unit tests in `tests/Shop.Api.Tests/Unit/SettingsTests.cs`.
- [ ] T010 [P] Domain in `shop/src/Shop.Api/Domain/`, enforcing these data-model.md fields:
  - `Item.id` (UUIDv7 generated by the app), `Order.createdAt` (UTC, set at placement);
  - `Item.name` (trimmed, 1–100, case-insensitive uniqueness key `nameKey`), `Item.stock` (≥ 0),
    `Item.priceCents` (≥ 1, decimal input ≤ 2 places, mUSDT cents);
  - `Order.reference` (10-char Crockford base32 from a CSPRNG), `Order.status` (`PendingPayment`
    only), `Order.walletAddress`, `Order.totalCents`, `Order.lines` (1–50, duplicates merged);
  - `OrderLine.quantity` (≥ 1), `OrderLine.unitPriceCents`, `OrderLine.name`.

  Unit tests in `tests/Shop.Api.Tests/Unit/DomainTests.cs`.
- [ ] T011 Store seam:
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
- [ ] T012 [P] `shop/src/Shop.Api/Storage/Postgres/` (`PostgresShopStore`, embedded numbered SQL
  scripts, `schema_version`, `pg_advisory_lock`):
  - order placement in one transaction with `FOR UPDATE` in id order;
  - `ILIKE` search with escaping;
  - pin the Testcontainers PostgreSQL image to the same major version as the CNPG operator's
    default image (read it from the installed CNPG release; note it in the log).

  Implements the PostgreSQL layout in data-model.md. `Integration/PostgresStoreTests.cs` subclasses
  the contract with a Testcontainers PostgreSQL container (same major as the CNPG default).
- [ ] T013 [P] `shop/src/Shop.Api/Storage/Redis/` (`RedisShopStore` and Lua scripts as embedded
  resources):
  - create, rename, delete and place-order scripts keep `shop:items`, `shop:item-names`,
    `shop:order:*` and `shop:orders` consistent;
  - `NX PX` init lock; `shop:schema`.

  Implements the Redis layout in data-model.md. `Integration/RedisStoreTests.cs` subclasses the
  contract with a Testcontainers Redis 7.2 container started with `--requirepass`.
- [ ] T014 Startup and auth (research R11, R12):
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

- [ ] T015 [US1] Admin endpoints in `shop/src/Shop.Api/Endpoints/Admin.cs` per
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
- [ ] T016 [P] [US1] Angular admin in `shop/src/Shop.Web/src/app/admin/`:
  - login page, auth guard, prompt to change the initial password;
  - items list with search, create/edit form with field errors, delete with confirm;
  - change-password page, sign-out.

---

## Phase 5: User Story 2 — Customer finds items and places an order (P1)

**Goal**: Search, cart and an atomic `PendingPayment` order (FR-009–FR-017).

**Independent Test**: The API tests below on both stores; quickstart Scenario 6.

- [ ] T017 [US2] Public endpoints in `shop/src/Shop.Api/Endpoints/Public.cs`:
  - `GET /api/shop`, `GET /api/items?q=&page=`;
  - `POST /api/orders` → 201 / 409 problems (nothing stored) / 400 / 503.

  `ApiTests.cs` covers US2 scenarios 1, 4, 5 and 6 on both stores: stock lowered by exactly the
  ordered quantities; `walletAddress` on the order equals `WALLET_ADDRESS`; no auth needed.
- [ ] T018 [P] [US2] Angular storefront in `shop/src/Shop.Web/src/app/store/`:
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

- [ ] T019 [US3] `GET /api/admin/orders?page=` in `shop/src/Shop.Api/Endpoints/Admin.cs`. `ApiTests.cs`
  on both stores covers:
  - 3 orders newest first;
  - 45 orders → pages of 20, 20 and 5 with each order exactly once;
  - snapshot name and price after the item is renamed or deleted;
  - 401 without a session.
- [ ] T020 [P] [US3] Angular orders page in `shop/src/Shop.Web/src/app/admin/orders/`: table with
  reference, time, status, total and expandable lines; pager.

---

## Phase 7: User Story 4 — Release and run on both modes (P2)

**Goal**: A released image runs on standard and light shops via the operator (FR-019–FR-026,
SC-003, SC-007–SC-009).

**Independent Test**: quickstart Scenarios 3–9 on the local cluster.

- [ ] T021 [US4] `shop/README.md`:
  - local run against Testcontainers or Docker Postgres/Redis;
  - the settings table (from contracts/operator-0.2.0.md);
  - tests, release steps, the admin-password flow and the lockout;
  - "do not paste the admin password anywhere".
- [ ] T022 [US4] Release:
  - push the branch when the maintainer asks, open the shop-spec PR, CI green, **ask before
    merging**;
  - tag `v0.1.0` and confirm the `release` job pushed `ghcr.io/nikola031ue/shop:0.1.0` within
    15 min (SC-008);
  - **ask before** making the GHCR package public; verify an anonymous `docker pull`.
- [ ] T026 Operator 0.2.1: self-healing Redis password (planned 2026-11-24; ID added after
  T025, runs here). Can start any time after Phase 1 and **must finish before T024**:
  - in `shop-operator/internal/controller/shop_controller.go` `prepareRedis`: when
    `<shop>-redis-auth` is missing and `<shop>-redis` exists, generate a new password Secret
    instead of reporting `RedisCredentialsLost`. Restart the Redis so it starts with the new
    password, keeping the PVC and its AOF data.
    - Design choice to settle in the PR: delete pod `<shop>-redis-0` (needs a new RBAC rule,
      pods `delete`) or another restart path the OT operator honours. Never delete the Redis CR
      or the PVC.
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
- [ ] T023 [US4] kube-state rollout:
  - `kube-state/clusters/local/shop-operator/values.yaml` adds
    `shopImage: {repository: ghcr.io/nikola031ue/shop, tag: "0.1.0"}`;
  - open the kube-state-spec PR, CI green, **ask before merging**;
  - run `./bring-up.sh` (only `shop-operator` upgraded); `std` and `lite` roll to the app and
    reach Ready.
- [ ] T024 [US4] Quickstart Scenarios 3–9 on Docker Desktop:
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
