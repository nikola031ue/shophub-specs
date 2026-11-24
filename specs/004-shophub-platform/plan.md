# Implementation Plan: ShopHub Platform (Requirements 1.1, 1.2, 3.3)

**Branch**: `004-shophub-platform` | **Date**: 2026-11-24 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/004-shophub-platform/spec.md`

## Summary

Two deliveries, strictly in order.

**Phase 1: platform** (no ShopHub app work before it is deployed).

- **Entry point**: kube-state gets a **Traefik** release (new upstream chart on the allowlist,
  research R1).
  - HTTPS on `localhost:443` through the Docker Desktop LoadBalancer; HTTP redirects to HTTPS.
  - A default TLS certificate from a hand-created Secret. It holds a **mkcert** wildcard for
    `*.shops.localhost` and `shophub.localhost`, so the maintainer's browser trusts it after
    `mkcert -install` (R2).
- **shop-operator 0.3.0**, CRDs unchanged:
  - Per Shop, an owned `Ingress` (host `<shop>.shops.localhost`, Shop status `url` becomes
    `https://…`).
  - Per Shop, an owned `NetworkPolicy`: only the entry point and ShopHub may reach the app.
  - A new env `TRUSTED_PROXY_CIDRS` for the app.
  - The operator watches **only the `shops` namespace**. The manager ClusterRole is replaced by a
    namespaced **Role in `shops`**, plus a by-name read of the Discord token in its own namespace
    (closes 002 R-1, R4).
  - Chart `shop-operator` 0.4.0 also creates the `shops` namespace.
- **shop 0.2.0**:
  - forwarded headers trusted only from `TRUSTED_PROXY_CIDRS`;
  - admin lockout per client address, with a shop-wide ceiling of 50 failures per 15 min;
  - order rate limit per real client;
  - session cookie always `Secure`;
  - `GET /api/shop` gains `adminPasswordChanged` for the FR-016 hand-off (R5, R6).

  Released `v0.2.0`, rolled out via kube-state.

**Then: the `shophub` app** (repo `shophub-spec`, folder `shophub`), built like `shop`:

- One image: an ASP.NET Core (.NET 10) minimal API serving a built Angular panel.
- Its own PostgreSQL (a CNPG `Cluster` in the `shophub` chart). Npgsql with hand-written SQL, as
  in `shop`.
- Owner accounts: email + password, cookie sessions, XSRF, lockout per (account, client
  address), Data-Protection keys in the database (R7).
- **Shop records** in its database are the source of truth. A background **reconciler** applies
  each record to the cluster with the official .NET Kubernetes client:
  - server-side apply of Shop, Wallet (registered only) and DiscordChannel in `shops`, field
    manager `shophub`;
  - deletes for removed shops;
  - a wallet switch: new Wallet `<id>-w<gen>`, repoint the Shop, delete the old one (R8, R9).
- Panel API with owner isolation in every query, proved by tests. The shop admin password is read
  on request from `<id>-admin` (R10).
- Tests:
  - xUnit with Testcontainers PostgreSQL;
  - Testcontainers **k3s** with the shop-operator CRDs, for the Kubernetes calls (R11);
  - `ng test`.
- The `shophub` chart replaces the placeholder: Deployment, Service, Ingress, ServiceAccount, a
  **Role + RoleBinding in `shops` only**, and the CNPG `Cluster`. Released and rolled out through
  kube-state.

## Technical Context

**Language/Version**:

- Go (shop-operator, from its `go.mod`).
- C# on .NET 10 (SDK in `global.json`, as in `shop`).
- TypeScript with Angular 22 (as in `shop`); Node 24 for builds.
- Bash and YAML for charts, kube-state and CI.

**Primary Dependencies**:

- Operator: controller-runtime (existing), `networking.k8s.io/v1` Ingress and NetworkPolicy.
- shop: unchanged stack plus ASP.NET Core `ForwardedHeaders`.
- shophub:
  - ASP.NET Core minimal APIs and Npgsql;
  - `Microsoft.Extensions.Identity.Core` (`PasswordHasher` only);
  - `KubernetesClient` (official .NET client: generic custom objects, server-side apply).
- Frontend: Angular router and HttpClient, no UI library.
- Platform: Traefik (upstream chart, pinned); mkcert on the maintainer's machine (Complexity
  Tracking).

**Storage**:

- ShopHub: PostgreSQL via CNPG (CNPG default major, matched in tests). Tables are in
  [data-model.md](data-model.md).
- Shops: unchanged (002/003).

**Testing**:

- Operator: envtest, extended for Ingress, NetworkPolicy and namespace scoping.
- shop: existing suites on both stores, plus forwarded-header, per-address lockout and
  `Secure`-cookie tests.
- shophub:
  - unit tests;
  - Testcontainers PostgreSQL (accounts, records, isolation);
  - Testcontainers k3s with the operator CRDs (reconciler, SSA, wallet switch, idempotent create
    and interrupted create);
  - API tests via `WebApplicationFactory`;
  - `ng test`.
- kube-state bats and e2e stay required. The chart `install` job covers the new `shophub` chart.

**Target Platform**: Docker Desktop Kubernetes; images `linux/amd64`; CI `ubuntu-latest`.

**Project Type**: web application (Angular + .NET in one image), plus operator and shop changes,
two chart changes, one new kube-state release and kube-state PRs.

**Performance Goals** (spec success criteria):

- Register → shop open within 5 min (SC-001).
- Shop in the list within 5 s; configuration in effect within 2 min (SC-002).
- Delete within 2 min (SC-003).
- Bring-up within 15 min; image within 15 min of the tag (SC-009).

**Constraints**:

- CRDs unchanged.
- ShopHub and the operator get namespaced Roles only, no ClusterRoles. Helm installs the CRDs,
  so the operator needs no cluster-wide access to them; it checks served database kinds through
  discovery, which every authenticated identity may read.
- No secrets in git, logs or status, except the FR-016 hand-off.
- The 5.3 layout is immutable.
- Solo developer, about 25 tasks.

**Scale/Scope**: tens of owners, 5 shops each (FR-011). Repos touched: `shop-operator`, `shop`,
`helm-charts` (`shop-operator` and `shophub` charts), `kube-state` (new `traefik` release, 3–4
PRs), and new code in `shophub`.

All items are resolved in [research.md](research.md); no NEEDS CLARIFICATION remain.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.* Constitution v1.2.1.

| Principle | Check | Pre-research | Post-design |
|-----------|-------|--------------|-------------|
| I. Immutable IaC layout | `charts/shophub` replaces the placeholder in place (layout of 5.3: `templates/` with deployment, service, ingress, serviceaccount, hpa, NOTES, helpers). `charts/shop-operator` gains the `shops` namespace and Role. kube-state adds `clusters/local/traefik/{helm.yaml,values.yaml}` and lists it in `cluster.yaml` `releaseOrder`, following the existing pattern | PASS | PASS |
| II. Repo-per-service, trunk-based | ShopHub lives in its own repo `shophub-spec`, protected before its first merge (ask first). Squash-only, Conventional Commits, `ghcr.io/nikola031ue/shophub:X.Y.Z` from `vX.Y.Z` on `main`, no `latest`. Operator and shop follow their existing paths | PASS | PASS |
| III. Tested before merge | ShopHub: unit tests, plus Testcontainers PostgreSQL and Testcontainers k3s (a real API server) as integration tests, required in CI. Operator: envtest. shop: both-store suites. Owner isolation has its own tests for every panel API (FR-018) | PASS | PASS |
| IV. Sepolia, tested locally | No chain interaction. Wallets are registered addresses only; no keys anywhere | PASS | PASS |
| V. Platform in git, shops via API | ShopHub is the only writer of Shop/Wallet/DiscordChannel, under its own ServiceAccount with a **Role in `shops` only**: exact kinds and verbs, plus Secret `get` used only for `<id>-admin`. The operator's ClusterRole becomes a Role in `shops` (closes 002 R-1). Ingress and NetworkPolicy are derived resources owned by the Shop. The `shops` namespace, Traefik and both charts are platform state in kube-state. User resources are never stored in git. The ShopHub DB record is the restore source (Principle V "restoring runtime state"). Hand-created TLS Secret and Phase 1 test Shops are tracked below | PASS (tracked) | PASS (tracked) |
| VI. Secrets in secret stores | Owner passwords hashed (PBKDF2); DB credentials via the CNPG Secret by reference; the TLS key only in a hand-created Secret, never in git (the mkcert CA key stays on the maintainer's machine). The shop admin password is read from its Secret on request only, never logged or cached (FR-016, the one documented hand-off) | PASS | PASS |
| VII. Per-Shop observability | Still not provisioned (spec 006). The `shophub` chart does not yet use the prometheus-stack that Requirement 3.3 and Principle V name | DEVIATION (recorded) | DEVIATION (recorded) |

Technology constraints:

- Angular and .NET are in the constitution's list.
- `KubernetesClient`, Npgsql and Testcontainers (k3s module) are libraries within the listed
  stack.
- **Traefik** (ingress controller) and **mkcert** (local CA) are new tools, justified in
  Complexity Tracking.

Gate result: **PASS**. Every exception is recorded in Complexity Tracking.

## Project Structure

### Documentation (this feature)

```text
specs/004-shophub-platform/
├── plan.md              # this file
├── research.md          # R1–R14
├── data-model.md        # ShopHub tables, resource naming, labels, state mapping
├── quickstart.md        # validation scenarios 1–9
├── contracts/
│   ├── platform-phase1.md   # entry point, operator 0.3.0, shop 0.2.0 deltas
│   ├── panel-api.md         # ShopHub REST API
│   └── delivery.md          # repo layout, CI, chart, RBAC, rollout order
├── checklists/requirements.md
└── tasks.md             # /speckit-tasks
```

### Source Code

```text
kube-state/clusters/local/traefik/{helm.yaml,values.yaml}   # new upstream release (Phase 1)
kube-state/clusters/local/cluster.yaml                     # releaseOrder: + traefik
kube-state/.github/upstream-charts.yaml                    # + traefik at its exact version
kube-state/bring-up.sh, README.md                          # TLS Secret precheck + one-time mkcert steps

shop-operator/                                  # 0.3.0 (Phase 1)
├── cmd/main.go                                 # --watch-namespace, --shop-domain, --ingress-class, --trusted-proxy-cidrs
├── internal/resources/{ingress,networkpolicy}.go, resources.go   # Ingress, NetworkPolicy, env, status URL
├── internal/controller/{shop_controller,config}.go               # Owns(Ingress, NetworkPolicy), namespaced cache
└── internal/controller/*_test.go, README.md

helm-charts/charts/shop-operator/   # 0.4.0: Namespace shops, Role+RoleBinding in shops, token Role, new flags
helm-charts/charts/shophub/         # 0.2.0: real app (replaces placeholder)

shop/                               # 0.2.0 (Phase 1)
├── src/Shop.Api/{Program,Settings,ForwardedHeaders}.cs, Auth/AdminAuth.cs, Endpoints/Public.cs
└── Storage/ (lockout per address + ceiling, both stores), tests

shophub/                            # new code, layout in contracts/delivery.md
├── src/ShopHub.Api/  (Program, Settings, Auth/, Accounts/, Shops/{ShopRecords, Reconciler, KubeGateway}, Endpoints/)
├── src/ShopHub.Web/  (Angular: register, sign-in, shop list, create, edit, delete, admin password)
├── tests/ShopHub.Api.Tests/ (Unit/, Integration/{Postgres, K3s, Api, Isolation})
├── Dockerfile, README.md, .github/workflows/ci.yaml
```

**Structure Decision**:

- Mirror `shop`: one API project, one Angular project, one test project.
- The Kubernetes calls sit behind one seam, `IKubeGateway` (apply, delete, read status, read the
  admin Secret). That way API tests can use a fake, and the k3s suite tests the real
  implementation.

## Delivery Notes (solo, ~25 tasks)

Estimated breakdown. `/speckit-tasks` will finalise it.

| Phase | Content | ≈ Tasks |
|-------|---------|---------|
| 1 Platform | Traefik release, TLS precheck and docs (kube-state PR); operator Ingress + status URL; NetworkPolicy + `TRUSTED_PROXY_CIDRS`; namespaced RBAC + watch-namespace + envtest; operator `v0.3.0` + chart 0.4.0 + kube-state PR; shop forwarded headers, `Secure` cookie, per-client limit, `adminPasswordChanged`; shop per-address lockout + ceiling on both stores; shop `v0.2.0` + kube-state PR + live Phase 1 checks (quickstart 1–3) | 8 |
| 2 ShopHub setup | Scaffold (solution, Angular, Dockerfile, CI); branch protection (ask first) | 2 |
| 3 Foundation | Settings, startup, DB migrations, Testcontainers harness; accounts (register, sign-in/out, lockout, sessions, XSRF); `IKubeGateway` with the k3s suite | 3 |
| 4 US2 accounts UI | Angular register / sign-in / guard | 1 |
| 5 US3 create and open | Shop records + reconciler (apply, status mapping, put back, idempotency); panel create/list/admin-password API with isolation tests; Angular list, create, open, show password | 3 |
| 6 US4/US5 change and delete | Change (availability, wallet switch) and delete API with tests; Angular edit and delete dialog | 2 |
| 7 Release | `shophub` chart; release `v0.1.0` (package public: ask first) + kube-state PR; quickstart 4–9 on the cluster | 3 |
| 8 Polish | `shophub` README; self-review and secrets sweep | 2 |
| | **Total** | **≈ 24** |

Within budget (~25, hard stop at ~30), so no split is needed.

Merges, GitHub settings changes and making a package public are each asked of the maintainer
first. Commits happen at the end of each phase; pushes only when asked.

## Complexity Tracking

| Violation / exception | Why Needed | Simpler Alternative Rejected Because |
|-----------------------|------------|--------------------------------------|
| New tool **Traefik** (ingress controller) as an upstream chart release | FR-020 needs an entry point. ingress-nginx is retired upstream (end of maintenance March 2026), so a maintained controller is needed. Traefik serves standard `Ingress`, so the operator stays controller-neutral | ingress-nginx: unmaintained. Gateway API + Envoy Gateway: more CRDs and moving parts for one cluster. The operator would also have to write `HTTPRoute`s instead of the standard `Ingress` |
| New tool **mkcert** on the maintainer's machine; TLS Secret `shophub-tls` created by hand in the `traefik` namespace (Principle V hand change, Principle VI secret) | FR-020: HTTPS the browser trusts, so `Secure` cookies work locally. The CA key must never be in git or the cluster | cert-manager + CA issuer: another operator, plus a chart for the issuer, and the browser still needs a one-time trust step. Traefik's default self-signed cert: a browser warning on every host. The Secret follows the documented pattern of the hand-created Discord token Secret; bring-up checks it exists |
| Hand changes for checks (Principle V): Phase 1 test Shops, Wallets and DiscordChannels in `shops`, deleted afterwards (quickstart 1–3); deleting one ShopHub-made Shop to prove drift repair (quickstart 8); the `traefik` namespace created by hand so the TLS Secret can exist before the release | ShopHub, the only allowed writer, does not exist until Phase 2; drift repair can only be shown by causing drift | Waiting leaves the operator and shop changes unverifiable on a real cluster. Automated tests use envtest and Testcontainers |
| Per-Shop observability not provisioned; the `shophub` chart doesn't use prometheus-stack yet (Principle VII, Requirement 3.3) | Carried over from 002/003. Observability is spec 006 | Wiring dashboards before the stack exists targets nothing; labels stay in place for 006 |
| Forwarded headers trusted from the pod CIDR, guarded by a NetworkPolicy (R5) | The app can't know the entry point's pod IPs. The NetworkPolicy makes sure only the entry point (and ShopHub) can reach a shop pod at all | A shared-secret header from Traefik would put a secret into a non-Secret resource. **Verified 2026-11-24:** Docker Desktop Kubernetes (v1.36.1, kind-based, CNI `kindnetd` v20260528) enforces NetworkPolicies. In a throwaway namespace, a deny-all Ingress policy blocked pod-to-pod HTTP; a `podSelector` allow rule let only the labelled client through; removing the policies restored access. `namespaceSelector` rules (as the Shop policy uses) are confirmed live in quickstart Scenario 2 |
