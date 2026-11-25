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

- [X] T001 [US1] Add the Traefik release to kube-state (research R1):
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
- [X] T002 [US1] TLS precheck and docs, then roll out 1A:
  - `kube-state/bring-up.sh`: before the `traefik` release, stop with the mkcert steps when
    Secret `shophub-tls` is missing in `traefik` (no secret value printed);
  - `kube-state/README.md` "Local HTTPS": `mkcert -install`, the cert for `shophub.localhost` and
    `*.shops.localhost`, the hand-made namespace and Secret (quickstart Prerequisites);
  - PR, CI green, **ask before merge**; the maintainer creates the Secret; `./bring-up.sh`;
  - check: `https://shophub.localhost` answers 404 from Traefik with a trusted certificate,
    `http://` redirects (quickstart Scenario 1 step 1); `traefik-proxies.traefik.svc.cluster.local`
    resolves to the Traefik pod IPs from a throwaway pod.

**Progress 1A (2026-11-24)**: T001 done; T002 code and docs done (exit 7 + mkcert help via
`cluster.yaml` `requiredSecrets`). kube-state PR #9 (branch `004-traefik`): all 7 checks green,
including e2e on kind. Squash-merged as kube-state `7bfa364`.

**Done 1A (2026-11-24)**: `./bring-up.sh` → `1 installed, 0 upgraded, 5 unchanged` (traefik 42.3.1).
Live from Windows:
- `http://shophub.localhost/` → 301 to `https://shophub.localhost/`;
- `https://shophub.localhost/` → 404 from Traefik;
- the served certificate is the mkcert one (SAN `shophub.localhost`, `*.shops.localhost`) and
  validates against the Windows trust store;
- `traefik-proxies` resolves to the Traefik pod IP `10.244.0.64`;
- the client address Traefik sees is `172.18.0.5`, the LoadBalancer proxy, not a pod or node
  (research R5 records this and its limitation).

Two notes for quickstart users on Windows:
- Windows `curl.exe` (Schannel) needs `--ssl-no-revoke`, because mkcert certificates have no
  revocation endpoint. Browsers don't.
- The Windows resolver doesn't resolve `*.localhost`. Browsers and curl resolve it themselves;
  PowerShell `Invoke-WebRequest` doesn't.

**Checkpoint 1A**: entry point live; nothing else changed.

### 1B — shop-operator `v0.3.0` (release: tag → chart 0.4.0 → kube-state)

- [X] T003 [P] [US1] Resource builders in `shop-operator/internal/resources/`:
  - new `ingress.go`: `Ingress <shop>` per data-model.md "Derived resources" (class, host
    `<shop>.<shop-domain>`, path `/` Prefix → Service `<shop>:80`, no `tls` block);
  - new `networkpolicy.go`: `NetworkPolicy <shop>` selecting `app.kubernetes.io/instance=<shop>`,
    ingress TCP 80 only from namespaces `traefik` and the ShopHub namespace;
  - `resources.go`: `ShopParams` gains domain, ingress class, trusted proxy host, ShopHub
    namespace; Deployment env `TRUSTED_PROXY_HOST` (research R5, revised 2026-11-25); status URL helper `https://<shop>.<shop-domain>`;
  - unit tests in `shop-operator/internal/resources/*_test.go`.
- [X] T004 [US1] Controller and manager in `shop-operator/`:
  - `cmd/main.go`: flags `--watch-namespace` (cache restricted to it; DiscordChannels and their
    Secrets also cached in the operator namespace, research R4), `--shop-domain`,
    `--ingress-class`, `--ingress-namespace`, `--trusted-proxy-host`, `--shophub-namespace`;
  - `api/v1alpha1/shop_types.go`: CRD rule rejecting a Shop name with a dot (FR-021), with an
    envtest validation case;
  - `internal/controller/shop_controller.go`: apply Ingress and NetworkPolicy (owned),
    `Owns(&Ingress{}, &NetworkPolicy{})`, `status.url` HTTPS; Discord token read by name with the
    uncached reader from the operator namespace;
  - RBAC markers plus `networking.k8s.io` ingresses and networkpolicies; `namespace=system`
    markers for the own-namespace Role; `hack/sync-chart.sh` writes Roles only;
  - envtest in `internal/controller/*_test.go`: children created, owned and restored after a
    manual edit; status URL; a Shop outside the watched namespace gets no children and no status;
    a DiscordChannel in the operator namespace is reconciled, one elsewhere is not;
    run via `optest.sh` in the dev container;
  - `README.md`: flags, RBAC, namespace.
- [X] T005 [US1] Chart `helm-charts/charts/shop-operator` 0.4.0:
  - `templates/namespace.yaml` (`shops`, label `shophub.io/shops: "true"`);
  - Role + RoleBinding in `shopsNamespace`, Role `…-platform-channels` in the release namespace
    (research R4), leader-election Role unchanged; ClusterRole/Binding removed;
  - values `shopsNamespace`, `shopDomain`, `ingressClassName`, `trustedProxyHost`,
    `shophubNamespace` wired to the flags; chart README and lint/template CI pass.
- [X] T006 [US1] Release and roll out 1B:
  - operator PR → **ask before merge** → tag `v0.3.0` → image;
  - helm-charts PR (chart 0.4.0, appVersion 0.3.1) → **ask before merge** → publish;
  - kube-state PR (`clusters/local/shop-operator/helm.yaml` 0.4.0; `shophub-discord`
    `targetNamespace: shop-operator-system`) → before merging, the hand migration in quickstart
    Scenario 1 step 1 (a, b) while 0.2.1 runs → **ask before merge** → `./bring-up.sh`;
  - live: quickstart Scenario 1 steps 2–4 (two hand-made test Shops in `shops`) and Scenario 2
    (`kubectl auth can-i` matrix, no ClusterRoleBinding, NetworkPolicy blocks `default` and allows
    `traefik`/`shophub`). Test Shops deleted afterwards (maintainer's decision, 2026-11-25); 1C
    creates its own.

**Progress 1B (2026-11-24)**:
- The platform alerts DiscordChannel moves to the operator namespace (decision recorded in
  research R4 and FR-026; migration order in quickstart Scenario 1 step 1).
- T003 and T004 are done: shop-operator PR #7 (branch `004-scoped-ingress`); `make lint` reports
  0 issues, the unit tests and envtest pass in the dev container, including the namespace
  scoping on a second API server.
- T005 chart 0.4.0 is committed locally in helm-charts (branch `004-shop-operator-0.4.0`). Its PR
  waits for the `v0.3.0` image, because CI installs the chart with its default `appVersion`.

**Progress 1B (2026-11-25)**:
- Before the merge, shops' forwarded-header trust was changed to Traefik only
  (`TRUSTED_PROXY_HOST`, research R5 revised) and the CRD now rejects dotted Shop names (FR-021).
- shop-operator PR #7 merged (`f2d8726`), tagged `v0.3.0`.
- The Shop NetworkPolicy's ShopHub peer was narrowed to the ShopHub app pods
  (`--shophub-pod-selector`, default `app.kubernetes.io/name=shophub,app.kubernetes.io/instance=shophub`): shop-operator PR #8 merged (`3d2e90f`), tagged
  `v0.3.1`, image published.
- T005 done: helm-charts PR #8 merged (`4bb9e0a`), chart `shop-operator` 0.4.0 with appVersion
  0.3.1 published; CI install job green.
- T006 done:
  - Hand migration while operator 0.2.1 ran: `platform-alerts` in `shophub` deleted in under 5 s
    (the finalizer removes itself only after the Discord delete succeeds); nothing named
    `platform-alerts` left in `shophub`; `helm uninstall shophub-discord -n shophub`.
  - kube-state PR #10 merged (`7a7615a`); bring-up: `1 installed, 1 upgraded, 4 unchanged`
    (shop-operator 0.4.0 upgraded, image 0.3.1; shophub-discord installed in
    `shop-operator-system`). `platform-alerts` Ready there with a new webhook Secret within
    15 s; Discord shows exactly one `platform-alerts` channel. Namespace `shops` created with
    label `shophub.io/shops=true`.
  - Scenario 1: test Shops `p1b-a`, `p1b-b` (registered Wallets, own DiscordChannels, database
    light) Ready within 30 s, status `url` `https://p1b-{a,b}.shops.localhost`. From Windows:
    HTTPS 200 with the certificate verified (Schannel `ssl_verify_result=0`), `http://` → 301 to
    `https://`, `/api/shop` returns `Phase 1B A` / `Phase 1B B`. Deleting `p1b-a`: its host
    returned 404 after 1 s while `p1b-b` kept 200. A dotted name (`dot.shop`) is rejected by the
    API server.
  - Scenario 2: `can-i get secrets` yes only in `shops` and `shop-operator-system`; no in
    `shophub`, `cnpg-system`, `traefik`, `default`; token by name yes; no cluster-wide list of
    Shops or namespaces; no ClusterRole or ClusterRoleBinding (Roles: manager in `shops`,
    platform-channels and leader-election in `shop-operator-system`). NetworkPolicy: from a
    throwaway pod in `default` the shop Service times out (000); from the ShopHub app pod
    `/api/shop` answers 200; through Traefik 200. The policy's ShopHub peer carries the pod
    selector. Deployment env `TRUSTED_PROXY_HOST=traefik-proxies.traefik.svc.cluster.local`
    (honoured from shop 0.2.0, 1C).
  - Pod selector check (2026-11-25, maintainer-approved): temporary Shop `np-check` in `shops`;
    a throwaway pod in `shophub` without ShopHub labels (`kubectl run --rm`, label
    `purpose=np-probe`) timed out against `http://np-check.shops.svc/api/shop` (000 after 5 s),
    while the ShopHub app pod got 200. The Shop, its DiscordChannel and Wallet were then deleted
    through the operator; `shops` is empty again.
  - Cleanup through the operator: both Shops, both DiscordChannels and both Wallets deleted;
    `shops` is empty; Discord shows no `p1b-*` channels, only `platform-alerts`.

**Checkpoint 1B**: shops reachable at `https://<shop>.shops.localhost`; R-1 closed (SC-007 for
the operator).

### 1C — shop `v0.2.0` (release: tag → kube-state)

- [X] T007 [P] [US1] Forwarded headers, cookies and order limit in `shop/src/Shop.Api/`:
  - `Settings.cs`: optional `TRUSTED_PROXY_HOST` (research R5);
  - new `ForwardedHeaders.cs`: `XForwardedFor | XForwardedProto`, trusted proxies are the
    addresses of `TRUSTED_PROXY_HOST` resolved by DNS every 30 s (same approach as ShopHub
    T013), `ForwardLimit = 1`; unset or failed lookup → no proxy trusted; wired in `Program.cs`;
  - session and anti-forgery cookies `CookieSecurePolicy.Always`;
  - order rate limit keyed by the resulting client address (`Endpoints/Public.cs`);
  - tests: header honoured from a resolved Traefik address, ignored from any other peer
    (including one inside the pod network, i.e. a ShopHub pod) and when the lookup fails; two addresses get
    separate order limits; `Secure` flag set.
- [X] T008 [P] [US1] Admin lockout per (account, client address) with shop-wide ceiling (FR-023),
  both stores, per data-model.md "shop app 0.2.0 storage additions":
  - `shop/src/Shop.Api/Storage/Scripts/003_lockout_per_account_address.sql` (`admin_lockouts`,
    PK `(account, address)`);
  - Redis `failed_signin.lua` updating `shop:admin:lock:<account>:<address>` and
    `shop:admin:lock:*` atomically, 15 min expiry;
  - store lockout API takes `(account, address)`; `Auth/AdminAuth.cs` passes account `admin`;
    refuse while either scope is locked (429 + `Retry-After`); success resets only its
    (account, address) scope; the order rate limit stays per address (T007);
  - tests on PostgreSQL and Redis: 5 failures lock that account from that address only; the same
    account from another address signs in; **5 failures for account `admin` from address A don't
    lock a different account from A** (store contract test with a second account name, since the
    app has one admin); 50 failures across addresses lock all; window expiry.
- [X] T009 [P] [US1] `GET /api/shop` returns `{name, adminPasswordChanged}` (false while the
  operator-generated password is in use, true after a change) in `shop/src/Shop.Api/Endpoints/`,
  with tests on both stores; README updated for T007–T009.
- [X] T010 [US1] Release and roll out 1C:
  - shop PR → **ask before merge** → tag `v0.2.0` → image;
  - kube-state PR (`clusters/local/shop-operator/values.yaml` `shopImage.tag: "0.2.0"`) → **ask
    before merge** → `./bring-up.sh`;
  - live: quickstart Scenario 3 through the Traefik Service from two throwaway pods; then delete
    the Phase 1 test Shops, Wallets, DiscordChannels and pods.

**Progress 1C (2026-11-25)**:
- T007–T009 done: shop PR #8 (branch `004-shop-0.2.0`), 208 tests passing on PostgreSQL and Redis
  (Testcontainers), `dotnet format` clean.
- Finding: with `CookieSecurePolicy.Always`, ASP.NET Core antiforgery throws on any plain-HTTP
  request. The Traefik-to-pod hop is HTTP, so a failed `TRUSTED_PROXY_HOST` lookup would have
  turned every admin request into a 500. Every cookie is now marked `Secure` by the cookie policy
  middleware instead; antiforgery keeps `SameAsRequest`. FR-025 still holds (tested over plain
  HTTP).
- The PostgreSQL migration file lives at `Storage/Postgres/Scripts/003_lockout_per_account_address.sql`
  (the repo's existing layout); stale `admin_lockouts` rows are pruned on each failure.
- Review before merge (maintainer): the shop-wide counter can't be fed by unknown account names
  (the login takes only a password; the account is always `admin`). But attempts that passed the
  lock check concurrently were still counted shop-wide after their (account, address) pair had
  locked, so one address could reach the 50-failure ceiling with a burst. Fixed: a failure while
  either scope is locked changes neither (`LockoutState.ApplyFailure`, `failed_signin.lua`).
  Tests: 60 concurrent failures from one address on three replicas leave the shop-wide count at
  5; the same burst over HTTP locks only that address; both failed against the old code. 215
  tests passing.
- shop PR #8 merged (`79a8ec3`), tagged `v0.2.0`, image `ghcr.io/nikola031ue/shop:0.2.0`
  published. kube-state PR #11 (`shopImage.tag: "0.2.0"`) open, CI green.
- T010 done:
  - kube-state PR #11 merged (`7d0e1e5`); bring-up: `0 installed, 1 upgraded, 5 unchanged`
    (shop-operator re-rendered with `--shop-image-tag=0.2.0`).
  - Test Shops `p1c-a` (database standard, PostgreSQL on CNPG, so migration 003 ran for real) and
    `p1c-b` (light, Redis), both Ready on `shop:0.2.0`, 2 replicas each. Probes: two throwaway
    pods (`10.244.0.95` pod A, `10.244.0.96` pod B) in throwaway namespace `p1c-probe`, over
    HTTPS through the Traefik Service (`--resolve` to its ClusterIP). The admin password went to
    the pods on stdin and was never printed.
  - Scenario 3 on `p1c-a`: (1) 5 wrong sign-ins from A → 401 ×5; right password from B → 204;
    right password from A → 429. (2) A with `X-Forwarded-For: 10.244.0.96` → still 429. The
    `admin_lockouts` table held `('admin', '10.244.0.95')` locked and `('*', '*')` at 5: A's two
    attempts while locked were not counted shop-wide. (3) A's orders: 201 ×20, then 429 (2
    replicas × 10, per-replica limit); B's order right after: 201. Quickstart step 3 corrected
    to 10 × replicas. (4) Cookie jar: `shop.session`, `shop.af`, `XSRF-TOKEN` all Secure (the
    first two HttpOnly). (5) `GET /api/shop`: `adminPasswordChanged` false, password change
    204, then true.
  - Browser check (maintainer, 2026-11-25): signed in at `https://p1c-b.shops.localhost/admin`
    with the generated password read in their own terminal; all three cookies Secure.
  - Cleanup through the operator: both Shops, DiscordChannels and Wallets deleted, namespace
    `p1c-probe` deleted; `shops` empty; Discord shows no `p1b-*`, `p1c-*` or `np-check`
    channels, only `platform-alerts`.

**Checkpoint Phase 1**: D-1 and R-1 closed; platform deployed. ShopHub work may start.

---

## Phase 2: ShopHub Setup

- [X] T011 Scaffold `shophub/` per contracts/delivery.md: `global.json`, `ShopHub.slnx`,
  `src/ShopHub.Api` (Program, `SecurityHeaders.cs`, `ErrorHandling.cs`, `/healthz`, SPA fallback),
  `src/ShopHub.Web` (Angular 22, router, HttpClient), `tests/ShopHub.Api.Tests` (xUnit v3,
  Microsoft.Testing.Platform), `Dockerfile` (node → sdk → chiseled aspnet, non-root, 8080),
  `.gitignore`, `.dockerignore`.
- [X] T012 CI in `shophub/.github/workflows/ci.yaml` (mirrors shop: `lint`, `test`, `release`
  with strict SemVer, tag-on-main and existing-version guards, SHA-pinned actions,
  `permissions: {}`); first PR; **ask before** setting branch protection (required `lint` +
  `test`, strict, linear, squash only, delete branch on merge).

**Progress Phase 2 (2026-11-25)**:
- T011–T012: shophub PR #1 (branch `004-scaffold`). Scaffold, Dockerfile (port 8080, non-root
  uid 1654, smoke-tested) and CI copied from the shop repo.
- Ported from shop 0.2.0 with their tests and adapted to many accounts (maintainer's request):
  trusted-proxy resolver and middleware; lockout per (account, address) plus a per-account
  ceiling (maintainer's choice) with the burst fix, PostgreSQL store `signin_lockouts` and the
  schema migrator; session check against per-session rows (`ISessionStore`), password change
  ending the owner's other sessions; cookie policy; registration rate limit. FR-002, FR-003,
  R7, data-model.md and panel-api.md (`POST /api/password`) amended. 60 .NET tests (PostgreSQL
  via Testcontainers), `ng lint`/`ng test`/`ng build` pass.
- Branch protection for `shophub-spec` set as in shop (maintainer-approved): required `lint` +
  `test`, strict, admins included, linear history, squash only, delete branch on merge. PR #1
  merged (`9f0e3c6`).

**Progress Phase 3 (2026-11-25)**:
- T013 done: shophub PR #2 (branch `004-foundation`). Settings (all five, problems named
  without values), `DatabaseInitializer` (from shop's `StoreInitializer`: migrations before the
  port opens, backoff capped at 10 s, redacted logs), `002_panel.sql` with the data-model rules
  as constraints, `/healthz` checks the database. 89 tests passing (Testcontainers PostgreSQL),
  including three replicas migrating at once and no secrets in logs when the database is
  unreachable.

---

## Phase 3: Foundation (blocks US2–US5)

- [X] T013 Settings, startup and schema in `shophub/src/ShopHub.Api/`:
  - `Settings.cs`: `DATABASE_URL`, `SHOPS_NAMESPACE`, `SHOP_DOMAIN`, `TRUSTED_PROXY_HOST`,
    `MAX_SHOPS_PER_OWNER` (contracts/delivery.md);
  - `ForwardedHeaders.cs` (ported in Phase 2; T013 only wires it with the other settings): trusted proxies limited to Traefik (research R7): the IPs of
    `TRUSTED_PROXY_HOST` resolved by DNS every 30 s; `X-Forwarded-For`/`-Proto` honoured only
    when the TCP peer is one of them, `ForwardLimit = 1`; unset or lookup failure means trust
    nobody (fail closed); unit tests with a fake resolver (trusted peer, other pod-network peer,
    failed lookup, address list change);
  - `Storage/Scripts/001_initial.sql` with the tables of data-model.md: `owners` (`owners.id`,
    `owners.email`, `owners.email_normalized` UNIQUE, `owners.password_hash`), `owner_sessions`
    (`owner_sessions.expires_at`), `shops` (`shops.id`, `shops.owner_id`,
    `shops.display_name`, `shops.display_name_key` with UNIQUE `(owner_id, display_name_key)`,
    `shops.availability`, `shops.database`, `shops.wallet_address`, `shops.wallet_generation`,
    `shops.state`, `shops.updated_at`), `data_protection_keys`, as script `002_panel.sql`
    (`001_signin_lockouts.sql` and the migration runner came with Phase 2);
  - Testcontainers PostgreSQL fixture in `tests/ShopHub.Api.Tests/Integration/Postgres/`.
- [X] T014 Kubernetes seam in `shophub/src/ShopHub.Api/Shops/`:
  - `IKubeGateway` + `KubeGateway` (official `KubernetesClient`): server-side apply with field
    manager `shophub`, delete, read Shop status, read Secret `<id>-admin` key `password`;
  - Testcontainers k3s fixture in `tests/ShopHub.Api.Tests/Integration/K3s/` loading the CRDs from
    the pinned `shop-operator` chart; the delivery.md Role applied to a test ServiceAccount;
  - tests: apply/delete/status round trip; CRD validation errors surface; the ServiceAccount is
    refused Secrets outside `shops`, kinds it doesn't own, and creating or changing a Secret or a
    Deployment inside `shops` (SC-007 for ShopHub).

- T014 done: shophub PR #3 (branch `004-kube-gateway`). `KubeGateway` (KubernetesClient 21.0.8):
  server-side apply with field manager `shophub`, Shop status, `<id>-admin` password, deletes one
  object at a time by names derived from the record (Shop, DiscordChannel, Wallets
  `-w1`…`-w<generation>`), never by label selector (maintainer's rule). k3s v1.36.1 via
  Testcontainers with the CRDs vendored from chart `shop-operator` 0.4.0
  (`tests/.../K3s/Manifests/`) and the delivery.md Role on a test ServiceAccount:
  SubjectAccessReview for every grant and refusal, real 403s, and the gateway's requests
  recorded (no selectors, no collection deletes, only Secret read `GET …/secrets/<id>-admin`).
  Finding: RBAC can't restrict `get secrets` to `<id>-admin` (`resourceNames` needs fixed names),
  so a known name like `<id>-db-app` is readable by the ServiceAccount; the gateway is the
  restriction, recorded in contracts/delivery.md. 127 tests passing (38 on k3s).
- The chart task must render the same Role as `tests/.../K3s/Manifests/shophub-rbac.yaml`;
  enforced by the T023 CI drift check.
- The Secret-read limit (any known Secret name in `shops` is readable; the gateway reads only
  `<id>-admin`) is accepted as documented by the maintainer (2026-11-25); no operator change.
- T015 done: shophub PR #4 (branch `004-accounts`). Accounts API, PostgreSQL stores for owners,
  sessions and Data-Protection keys, the Phase 2 lockout and rate limit wired in. No enumeration
  as specified; the counting hasher found that a lazily built dummy hash made the first
  unknown-email sign-in do an extra PBKDF2 run, fixed by building it when the service is created.
  147 tests passing.

**Checkpoint**: app starts against PostgreSQL; gateway proven against a real API server.

---

## Phase 4: User Story 2 — Register and sign in (Priority: P1)

**Goal**: accounts with sessions, lockout and sign-out ([contracts/panel-api.md](contracts/panel-api.md) Accounts).

**Independent test**: quickstart Scenario 5.

- [X] T015 [US2] Accounts backend in `shophub/src/ShopHub.Api/{Accounts,Auth,Endpoints/Accounts.cs}`:
  - `GET /api/session`, `POST /api/register`, `/api/signin`, `/api/signout`, `/api/password`;
  - validation per data-model.md `owners` (email, password length); `PasswordHasher` into
    `owners.password_hash`; **no account enumeration** (FR-001, panel-api.md): register answers
    202 with no body and no cookie for both a new and an existing `owners.email_normalized`
    (existing account untouched; one hash computed either way); sign-in with an unknown email
    gives the wrong-password 401 and verifies against a fixed dummy hash;
  - sessions per `owner_sessions` (8 h, row checked on every request, deleted on sign-out; a
    password change deletes the owner's other rows): a PostgreSQL `ISessionStore` for the Phase 2
    `Auth/Sessions.cs`;
  - lockout: wire the Phase 2 `SignInLockout` + `PostgresLockoutStore` (data-model.md
    `signin_lockouts`, per (account, address) plus the per-account ceiling), 429 +
    `Retry-After`; rate limits stay per client address: registration 10/hour (Phase 2
    `RateLimits.Registration`);
  - session cookie `Secure`, `HttpOnly`, `SameSite=Strict` (FR-004); XSRF cookie/header;
    Data-Protection keys in `data_protection_keys`;
  - tests (unit + Postgres + `WebApplicationFactory`): FR-001–FR-004 including the three cookie
    flags, replayed cookie after sign-out → 401, **after a password change a stolen cookie of
    another session → 401 while the current session stays**, lockout from one address does not
    block the same account from another, and **5 failures for account X from address A don't lock
    account Y from address A**; 50 across addresses lock X but not Y; **no enumeration**: an
    existing-email registration and a new one return identical status, headers (no
    `Set-Cookie` session) and body, and an unknown-email sign-in and a wrong-password sign-in
    return identical status and body. Timing, robust against CI noise: the assertion is on hash
    cost, not wall clock. A counting `IPasswordHasher` wrapper registered in the test host shows
    every path does exactly one PBKDF2 operation (hash or verify, same iteration count): new and
    existing registration, wrong password and unknown email. Wall clock is only a coarse guard:
    the median over 20 requests each is within a factor of 5.
- [X] T016 [P] [US2] Angular auth in `shophub/src/ShopHub.Web/src/app/auth/`: register and
  sign-in pages (after a 202 registration the page signs in with the same credentials; a 401
  there shows "If you already have an account, sign in with its password"), XSRF interceptor,
  route guard, sign-out; `ng test`. The register page's failure wording reveals no more than the
  API: never "email taken" or "account exists"; tested (exact text, forbidden words, a new and a
  taken email give the same page).

Progress (Phase 4):

- T015 merged: shophub PR #4 (718c554).
- T016 merged: shophub PR #5 (d1bad2b). Register, sign-in, sign-out, owner and guest
  guards, own XSRF interceptor (fetches a missing token, replaces a stale one once on the API's
  XSRF 400; other `/api` 401s go to sign-in). 29 `ng test` tests; a mutation to "This email is
  already taken." fails them. `compose.yaml` runs PostgreSQL plus the image at
  http://localhost:8080 for a browser click-through (passwordless trust database, no published
  DB port). Smoke test against the real API in compose matched the contract. Browser
  click-through by the maintainer (2026-11-25, `docker compose` at http://localhost:8080): OK.

**Checkpoint**: Scenario 5 passes locally.

---

## Phase 5: User Story 3 — Create and open a shop (Priority: P1) 🎯 MVP

**Goal**: an owner creates a shop, sees its state, opens it, gets the admin password.

**Independent test**: quickstart Scenarios 6 and 7.

- [X] T017 [US3] Shop records in `shophub/src/ShopHub.Api/Shops/ShopRecords.cs`:
  - id generation per data-model.md `shops.id`; name rules per `shops.display_name` and
    `shops.display_name_key`; `shops.availability`, `shops.database` enums; `shops.wallet_address`
    format and EIP-55 checksum;
  - insert in one transaction with the per-owner limit under a row lock on the owner (FR-011);
    every owner-facing query filters on `owner_id`;
  - idempotent create (double submit → one shop, SC-010);
  - unit + Postgres tests.

  Progress: done in shophub PR #6 (branch `004-shop-records`). Identical repeat returns the
  existing shop, other settings with the same name are a duplicate, repeat check before the limit;
  wallet stored in EIP-55 form; names refuse control characters. Keccak-256 in-house (.NET SHA3 has
  the wrong padding), cross-checked against platform SHA3-256. 210 tests; the two concurrency tests
  (12 creates from 3 replicas) fail without the owner row lock. The example id `s7k2m9qxa1` in
  panel-api.md and R9 was not valid base32 and is now `s7k2m4qxab`.
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
  - **Role drift check (maintainer, 2026-11-25):** CI renders the chart's `rbac.yaml` (`helm
    template` with default values) and compares the Role and RoleBinding with
    `shophub/tests/ShopHub.Api.Tests/Integration/K3s/Manifests/shophub-rbac.yaml`, the manifest
    the k3s RBAC tests assert. Rules, subjects and namespace are compared after normalising (YAML to
    JSON, rule lists sorted; labels and annotations ignored). Any difference fails the job, so the
    deployed RBAC can't drift from the tested one.
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
