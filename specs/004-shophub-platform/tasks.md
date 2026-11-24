---

description: "Task list for 004-shophub-platform"
---

# Tasks: ShopHub Platform (Requirements 1.1, 1.2, 3.3)

**Input**: Design documents from `/specs/004-shophub-platform/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Required. Constitution Principle III: unit and integration tests before merge (envtest
for the operator, Testcontainers PostgreSQL/Redis for shop, Testcontainers PostgreSQL and k3s for
ShopHub). FR-018 requires owner-isolation tests for every panel API.

**Organization**:

- **Phase 1 is the platform (US1)** in three groups. Each group ends with its own release and
  rollout: **1A** Traefik + mkcert (kube-state), **1B** shop-operator `v0.3.0` (chart 0.4.0),
  **1C** shop `v0.2.0`. No ShopHub task starts before 1C is deployed and checked (spec
  Dependencies).
- ShopHub phases follow: Setup → Foundation → US2 → US3 → US4 → US5 → Release → Polish.
- Every merge, GitHub settings change and package visibility change is asked of the maintainer
  first. Commits are made at the end of each phase. Nothing is pushed unless the maintainer asks.
- Hand cluster changes only where recorded in plan.md Complexity Tracking.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: User story the task belongs to (US1–US5)
- Paths: `kube-state/…`, `shop-operator/…`, `shop/…`, `helm-charts/…` are the existing sibling
  repos; `shophub/…` is the app repo (layout in [contracts/delivery.md](contracts/delivery.md))

---

## Phase 1: Platform (US1, Priority P1) — blocks all ShopHub work

**Goal**: every shop has its own HTTPS address; shops see the real client address; the operator
has namespaced permissions only ([contracts/platform-phase1.md](contracts/platform-phase1.md)).

**Independent test**: quickstart Scenarios 1–3.

### 1A — Entry point: Traefik + mkcert (release: kube-state PR)

- [ ] T001 [US1] Add the Traefik release to kube-state (research R1):
  - `kube-state/clusters/local/traefik/helm.yaml` (upstream OCI chart, exact pinned version) and
    `values.yaml` (namespace `traefik`, `LoadBalancer`, `externalTrafficPolicy: Local`, ports
    80/443, permanent `web` → `websecure` redirect, IngressClass `traefik` as default, default
    certificate from Secret `shophub-tls`);
  - `kube-state/clusters/local/cluster.yaml` `releaseOrder`: `traefik` after `redis-operator`,
    before `shop-operator`;
  - headless Service `traefik-proxies` (`clusterIP: None`, selecting the Traefik pods) through the
    chart's `extraObjects`, so ShopHub can resolve the Traefik pod IPs by DNS
    (contracts/platform-phase1.md, research R7);
  - `kube-state/.github/upstream-charts.yaml`: allowlist entry at the exact version;
  - bats cases for the new release in `kube-state/.github/tests/*.bats`.
- [ ] T002 [US1] TLS precheck and docs, then roll out 1A:
  - `kube-state/bring-up.sh`: before the `traefik` release, stop with the mkcert steps when
    Secret `shophub-tls` is missing in `traefik` (no secret value printed);
  - `kube-state/README.md` "Local HTTPS": `mkcert -install`, the cert for `shophub.localhost` and
    `*.shops.localhost`, the hand-made namespace and Secret (quickstart Prerequisites);
  - PR, CI green, **ask before merge**; the maintainer creates the Secret; `./bring-up.sh`;
  - check: `https://shophub.localhost` answers 404 from Traefik with a trusted certificate,
    `http://` redirects (quickstart Scenario 1 step 1); `traefik-proxies.traefik.svc.cluster.local`
    resolves to the Traefik pod IPs from a throwaway pod.

**Checkpoint 1A**: entry point live; nothing else changed.

### 1B — shop-operator `v0.3.0` (release: tag → chart 0.4.0 → kube-state)

- [ ] T003 [P] [US1] Resource builders in `shop-operator/internal/resources/`:
  - new `ingress.go`: `Ingress <shop>` per data-model.md "Derived resources" (class, host
    `<shop>.<shop-domain>`, path `/` Prefix → Service `<shop>:80`, no `tls` block);
  - new `networkpolicy.go`: `NetworkPolicy <shop>` selecting `app.kubernetes.io/instance=<shop>`,
    ingress TCP 80 only from namespaces `traefik` and the ShopHub namespace;
  - `resources.go`: `ShopParams` gains domain, ingress class, trusted CIDRs, ShopHub namespace;
    Deployment env `TRUSTED_PROXY_CIDRS`; status URL helper `https://<shop>.<shop-domain>`;
  - unit tests in `shop-operator/internal/resources/*_test.go`.
- [ ] T004 [US1] Controller and manager in `shop-operator/`:
  - `cmd/main.go`: flags `--watch-namespace` (cache restricted to it), `--shop-domain`,
    `--ingress-class`, `--trusted-proxy-cidrs`, `--shophub-namespace`;
  - `internal/controller/shop_controller.go`: apply Ingress and NetworkPolicy (owned),
    `Owns(&Ingress{}, &NetworkPolicy{})`, `status.url` HTTPS; Discord token read by name with the
    uncached reader from the operator namespace;
  - RBAC markers → namespaced Role (`config/rbac/`), plus `networking.k8s.io` ingresses and
    networkpolicies;
  - envtest in `internal/controller/*_test.go`: children created, owned and restored after a
    manual edit; status URL; a Shop outside the watched namespace gets no children and no status;
    run via `optest.sh` in the dev container;
  - `README.md`: flags, RBAC, namespace.
- [ ] T005 [US1] Chart `helm-charts/charts/shop-operator` 0.4.0:
  - `templates/namespace.yaml` (`shops`, label `shophub.io/shops: "true"`);
  - Role + RoleBinding in `shopsNamespace`, Role `shop-operator-discord-token` (`resourceNames`)
    in the release namespace, leader-election Role unchanged; ClusterRole/Binding removed;
  - values `shopsNamespace`, `shopDomain`, `ingressClassName`, `trustedProxyCIDRs`,
    `shophubNamespace` wired to the flags; chart README and lint/template CI pass.
- [ ] T006 [US1] Release and roll out 1B:
  - operator PR → **ask before merge** → tag `v0.3.0` → image;
  - helm-charts PR (chart 0.4.0, appVersion 0.3.0) → **ask before merge** → publish;
  - kube-state PR (`clusters/local/shop-operator/helm.yaml` 0.4.0) → **ask before merge** →
    `./bring-up.sh`;
  - live: quickstart Scenario 1 steps 2–4 (two hand-made test Shops in `shops`) and Scenario 2
    (`kubectl auth can-i` matrix, no ClusterRoleBinding, NetworkPolicy blocks `default` and allows
    `traefik`/`shophub`). Keep the test Shops for 1C.

**Checkpoint 1B**: shops reachable at `https://<shop>.shops.localhost`; R-1 closed (SC-007 for
the operator).

### 1C — shop `v0.2.0` (release: tag → kube-state)

- [ ] T007 [P] [US1] Forwarded headers, cookies and order limit in `shop/src/Shop.Api/`:
  - `Settings.cs`: optional `TRUSTED_PROXY_CIDRS` (comma-separated); invalid CIDR stops start with
    a message naming the variable;
  - new `ForwardedHeaders.cs`: `XForwardedFor | XForwardedProto`, `KnownNetworks` from the
    setting only, `ForwardLimit = 1`; unset → headers ignored; wired in `Program.cs`;
  - session and anti-forgery cookies `CookieSecurePolicy.Always`;
  - order rate limit keyed by the resulting client address (`Endpoints/Public.cs`);
  - tests: header honoured from a trusted peer, ignored from an untrusted one; two addresses get
    separate order limits; `Secure` flag set.
- [ ] T008 [P] [US1] Per-address admin lockout with shop-wide ceiling (FR-023), both stores, per
  data-model.md "shop app 0.2.0 storage additions":
  - `shop/src/Shop.Api/Storage/Scripts/003_lockout_per_address.sql` (`admin_lockouts`);
  - Redis `failed_signin.lua` updating `shop:admin:lock:<address>` and `shop:admin:lock:*`
    atomically, 15 min expiry;
  - `Auth/AdminAuth.cs`: refuse while either scope is locked (429 + `Retry-After`); success resets
    only its address scope;
  - tests on PostgreSQL and Redis: 5 failures lock one address only; another address signs in;
    50 failures across addresses lock all; window expiry.
- [ ] T009 [P] [US1] `GET /api/shop` returns `{name, adminPasswordChanged}` (false while the
  operator-generated password is in use, true after a change) in `shop/src/Shop.Api/Endpoints/`,
  with tests on both stores; README updated for T007–T009.
- [ ] T010 [US1] Release and roll out 1C:
  - shop PR → **ask before merge** → tag `v0.2.0` → image;
  - kube-state PR (`clusters/local/shop-operator/values.yaml` `shopImage.tag: "0.2.0"`) → **ask
    before merge** → `./bring-up.sh`;
  - live: quickstart Scenario 3 through the Traefik Service from two throwaway pods; then delete
    the Phase 1 test Shops, Wallets, DiscordChannels and pods.

**Checkpoint Phase 1**: D-1 and R-1 closed; platform deployed. ShopHub work may start.

---

## Phase 2: ShopHub Setup

- [ ] T011 Scaffold `shophub/` per contracts/delivery.md: `global.json`, `ShopHub.slnx`,
  `src/ShopHub.Api` (Program, `SecurityHeaders.cs`, `ErrorHandling.cs`, `/healthz`, SPA fallback),
  `src/ShopHub.Web` (Angular 22, router, HttpClient), `tests/ShopHub.Api.Tests` (xUnit v3,
  Microsoft.Testing.Platform), `Dockerfile` (node → sdk → chiseled aspnet, non-root, 8080),
  `.gitignore`, `.dockerignore`.
- [ ] T012 CI in `shophub/.github/workflows/ci.yaml` (mirrors shop: `lint`, `test`, `release`
  with strict SemVer, tag-on-main and existing-version guards, SHA-pinned actions,
  `permissions: {}`); first PR; **ask before** setting branch protection (required `lint` +
  `test`, strict, linear, squash only, delete branch on merge).

---

## Phase 3: Foundation (blocks US2–US5)

- [ ] T013 Settings, startup and schema in `shophub/src/ShopHub.Api/`:
  - `Settings.cs`: `DATABASE_URL`, `SHOPS_NAMESPACE`, `SHOP_DOMAIN`, `TRUSTED_PROXY_HOST`,
    `MAX_SHOPS_PER_OWNER` (contracts/delivery.md);
  - `ForwardedHeaders.cs`: trusted proxies limited to Traefik (research R7): the IPs of
    `TRUSTED_PROXY_HOST` resolved by DNS every 30 s; `X-Forwarded-For`/`-Proto` honoured only
    when the TCP peer is one of them, `ForwardLimit = 1`; unset or lookup failure means trust
    nobody (fail closed); unit tests with a fake resolver (trusted peer, other pod-network peer,
    failed lookup, address list change);
  - `Storage/Scripts/001_initial.sql` with the tables of data-model.md: `owners` (`owners.id`,
    `owners.email`, `owners.email_normalized` UNIQUE, `owners.password_hash`), `owner_sessions`
    (`owner_sessions.expires_at`), `signin_failures` (`signin_failures.email_normalized`,
    `signin_failures.address`, `signin_failures.at`), `shops` (`shops.id`, `shops.owner_id`,
    `shops.display_name`, `shops.display_name_key` with UNIQUE `(owner_id, display_name_key)`,
    `shops.availability`, `shops.database`, `shops.wallet_address`, `shops.wallet_generation`,
    `shops.state`, `shops.updated_at`), `data_protection_keys`; migration runner as in shop;
  - Testcontainers PostgreSQL fixture in `tests/ShopHub.Api.Tests/Integration/Postgres/`.
- [ ] T014 Kubernetes seam in `shophub/src/ShopHub.Api/Shops/`:
  - `IKubeGateway` + `KubeGateway` (official `KubernetesClient`): server-side apply with field
    manager `shophub`, delete, read Shop status, read Secret `<id>-admin` key `password`;
  - Testcontainers k3s fixture in `tests/ShopHub.Api.Tests/Integration/K3s/` loading the CRDs from
    the pinned `shop-operator` chart; the delivery.md Role applied to a test ServiceAccount;
  - tests: apply/delete/status round trip; CRD validation errors surface; the ServiceAccount is
    refused Secrets outside `shops`, kinds it doesn't own, and creating or changing a Secret or a
    Deployment inside `shops` (SC-007 for ShopHub).

**Checkpoint**: app starts against PostgreSQL; gateway proven against a real API server.

---

## Phase 4: User Story 2 — Register and sign in (Priority: P1)

**Goal**: accounts with sessions, lockout and sign-out ([contracts/panel-api.md](contracts/panel-api.md) Accounts).

**Independent test**: quickstart Scenario 5.

- [ ] T015 [US2] Accounts backend in `shophub/src/ShopHub.Api/{Accounts,Auth,Endpoints/Accounts.cs}`:
  - `GET /api/session`, `POST /api/register`, `/api/signin`, `/api/signout`;
  - validation per data-model.md `owners` (email, password length); `PasswordHasher` into
    `owners.password_hash`; duplicate by `owners.email_normalized` → 409;
  - sessions per `owner_sessions` (8 h, row checked on every request, deleted on sign-out);
  - lockout per `signin_failures` `(email_normalized, address)`, 429 + `Retry-After`;
    registration limit 10/hour per client address;
  - session cookie `Secure`, `HttpOnly`, `SameSite=Strict` (FR-004); XSRF cookie/header;
    Data-Protection keys in `data_protection_keys`;
  - tests (unit + Postgres + `WebApplicationFactory`): FR-001–FR-004 including the three cookie
    flags, replayed cookie after sign-out → 401, lockout from one address does not block
    another.
- [ ] T016 [P] [US2] Angular auth in `shophub/src/ShopHub.Web/src/app/auth/`: register and
  sign-in pages, XSRF interceptor, route guard, sign-out; `ng test`.

**Checkpoint**: Scenario 5 passes locally.

---

## Phase 5: User Story 3 — Create and open a shop (Priority: P1) 🎯 MVP

**Goal**: an owner creates a shop, sees its state, opens it, gets the admin password.

**Independent test**: quickstart Scenarios 6 and 7.

- [ ] T017 [US3] Shop records in `shophub/src/ShopHub.Api/Shops/ShopRecords.cs`:
  - id generation per data-model.md `shops.id`; name rules per `shops.display_name` and
    `shops.display_name_key`; `shops.availability`, `shops.database` enums; `shops.wallet_address`
    format and EIP-55 checksum;
  - insert in one transaction with the per-owner limit under a row lock on the owner (FR-011);
    every owner-facing query filters on `owner_id`;
  - idempotent create (double submit → one shop, SC-010);
  - unit + Postgres tests.
- [ ] T018 [US3] Reconciler in `shophub/src/ShopHub.Api/Shops/{Reconciler,StateMapping,ReasonTexts}.cs`:
  - hosted service, every 30 s plus a trigger after each write, PostgreSQL advisory lock;
  - applies DiscordChannel `<id>`, Wallet `<id>-w<generation>` (from `shops.wallet_generation`)
    and Shop `<id>` with the labels and specs of data-model.md "Cluster resources";
  - puts back missing resources (drift); finishes an interrupted create;
  - works by name from its records only: it never lists-and-prunes by label, so resources
    without a record are never touched (FR-015);
  - panel state per data-model.md "Panel state mapping" (`shops.state`, `shops.updated_at`) and
    reason texts per panel-api.md;
  - k3s tests: create; drift repair; interrupted create at each step (after the DiscordChannel,
    after the Wallet, after the Shop) finishes with no orphan (SC-010); repeated apply is a
    no-op; a hand-made Shop carrying ShopHub labels but no record survives every pass.
- [ ] T019 [US3] Shop endpoints in `shophub/src/ShopHub.Api/Endpoints/Shops.cs`:
  - `GET /api/shops`, `POST /api/shops`, `GET /api/shops/{id}`,
    `POST /api/shops/{id}/admin-password` (status read over the in-cluster Service first; 409
    `changed` / `notReady`; `Cache-Control: no-store`; password never logged);
  - 503 when the database or Kubernetes API is down;
  - API tests plus `tests/ShopHub.Api.Tests/Integration/Isolation/` suite: owner B gets 404 for
    every endpoint on owner A's shop, and A's shops are absent from B's list (FR-018, SC-004).
- [ ] T020 [P] [US3] Angular shops in `shophub/src/ShopHub.Web/src/app/shops/`:
  - list with states and reasons, refreshed at least every 10 s while any shop is not ready
    (FR-008);
  - create form with field errors and the limit message;
  - open in a new tab with `target="_blank" rel="noopener noreferrer"` (FR-010);
  - "Show shop admin password" with the hint to change it at the first shop sign-in (FR-016),
    and the "password was changed" message;
  - `ng test` covering the refresh interval, the `rel` attribute and the hint.

**Checkpoint (MVP)**: register → create → open works locally against k3s/fake gateway.

---

## Phase 6: User Story 4 — Change configuration (Priority: P2)

**Independent test**: quickstart Scenario 8 steps 1–2.

- [ ] T021 [US4] `PATCH /api/shops/{id}` in `Endpoints/Shops.cs` + reconciler + Angular edit
  (`src/app/shops/edit`):
  - availability change; wallet change increments `shops.wallet_generation` (data-model.md), the
    reconciler applies the new Wallet, repoints the Shop, then deletes the old Wallet (FR-009);
  - 400 for `name`/`database`, 409 while deleting;
  - k3s test for the wallet switch order; API and isolation tests for PATCH.

---

## Phase 7: User Story 5 — Delete a shop (Priority: P2)

**Independent test**: quickstart Scenario 8 steps 3–4.

- [ ] T022 [US5] `DELETE /api/shops/{id}` in `Endpoints/Shops.cs` + reconciler + Angular delete
  dialog (`src/app/shops/delete`):
  - `confirmName` must match (trimmed, exact case) else 400; `shops.state` → `Deleting` (202);
  - reconciler deletes Shop, Wallet and DiscordChannel by name from the record (never by label
    selector), removes the row once all are gone; repeat-safe;
  - k3s, API and isolation tests for DELETE.

---

## Phase 8: Release (FR-027, FR-028, FR-029)

- [ ] T023 Chart `helm-charts/charts/shophub` 0.2.0 per contracts/delivery.md (deployment,
  service, ingress, serviceaccount, `rbac.yaml` Role + RoleBinding in `shopsNamespace` only,
  `database.yaml` CNPG `shophub-db`, `networkpolicy.yaml` (ShopHub app pods accept TCP 8080 only
  from the Traefik pods in namespace `traefik`; the database pods stay reachable from the app),
  value `trustedProxyHost`, hpa disabled, NOTES, helpers); helm-charts PR, chart `install` job
  green, **ask before merge**.
- [ ] T024 Release and roll out ShopHub:
  - shophub PR(s) → **ask before merge** → tag `v0.1.0` → image; **ask before** making the
    package public;
  - publish chart 0.2.0 (appVersion 0.1.0);
  - kube-state PR `clusters/local/shophub/{helm.yaml,values.yaml}` → **ask before merge** →
    `./bring-up.sh`; quickstart Scenario 4; record the tag-to-pullable-image time (SC-009).
- [ ] T025 Live validation: quickstart Scenario 4 full run from an empty cluster (bring-up under
  15 min, FR-028, SC-009), then Scenarios 5–9; record timings (SC-001–SC-003, SC-009) and the
  secret sweep (SC-008) in this file; clean up test owners' shops via the panel.

---

## Phase 9: Polish

- [ ] T026 `shophub/README.md` (settings, local run, tests, release); self-review of all 004 PRs;
  set spec.md status to Implemented after the maintainer's OK.

---

## Dependencies & Execution Order

```text
1A (T001–T002) ─► 1B (T003–T006) ─► 1C (T007–T010) ─► Setup (T011–T012) ─► Foundation (T013–T014)
                                                                              │
                                     US2 (T015–T016) ◄────────────────────────┤
                                     US3 (T017–T020) ◄── needs T015 sessions ─┘
                                     US4 (T021), US5 (T022) ◄── need US3
                                     Release (T023–T025) ◄── all stories
                                     Polish (T026)
```

- 1C code (T007–T009) may be written while 1B is in review; its rollout (T010) needs 1B live.
- Inside a group, the release task is last.

## Parallel Opportunities

- 1B: T003 alongside T005 (different repos).
- 1C: T007, T008, T009 (different files; merge conflicts only in README, resolve at the end).
- US2: T016 alongside T015 (contract fixed in panel-api.md).
- US3: T020 alongside T017–T019.
- US4 and US5 touch the same files; run them one after the other.

## Implementation Strategy

1. Phase 1 groups in order, each released and checked live before the next rollout.
2. ShopHub Setup + Foundation, then US2 + US3 = MVP (register, create, open).
3. US4, US5, then release and live validation.
4. Commit at each phase end; push only on request.

**Total: 26 tasks** (US1 10, Setup 2, Foundation 2, US2 2, US3 4, US4 1, US5 1, Release 3,
Polish 1). Within the ~25 budget and below the ~30 split threshold.
