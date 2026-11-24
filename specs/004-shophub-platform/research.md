# Research: ShopHub Platform (004)

Each entry: Decision, Rationale, Alternatives. Versions are pinned at task time to the latest
stable release that is at least two weeks old.

## R1 — Cluster entry point (FR-020)

- **Decision**: **Traefik** v3 via its upstream Helm chart (OCI, `ghcr.io/traefik/helm/traefik`),
  as a new kube-state release `traefik` in namespace `traefik`. The chart's exact version is
  added to `.github/upstream-charts.yaml`.
  - The Service is a `LoadBalancer` with `externalTrafficPolicy: Local`, so Docker Desktop
    publishes it on `localhost:80/443`.
  - Entry point `web` redirects to `websecure`.
  - The IngressClass is `traefik`, the default class.
  - Traefik sets `X-Forwarded-For`, `X-Forwarded-Proto` and `X-Forwarded-Host`, and strips any
    client-supplied values. The chart's `forwardedHeaders.trustedIPs` stays empty.
- **Rationale**: a maintained controller that serves standard `networking.k8s.io/v1` `Ingress`,
  so the operator writes a portable resource. ingress-nginx reached end of maintenance in March
  2026.
- **Alternatives**: ingress-nginx (unmaintained); Gateway API with Envoy Gateway (more CRDs; the
  operator would write `HTTPRoute`); NGINX Inc. controller (separate CRDs for the features we'd
  use).

## R2 — Local HTTPS that the browser trusts (FR-020, FR-025)

- **Decision**: **mkcert** on the maintainer's machine.
  - `mkcert -install` once trusts its local CA.
  - `mkcert shophub.localhost '*.shops.localhost'` makes one certificate.
  - It goes into Secret `shophub-tls` in namespace `traefik` (`kubectl create secret tls`, hand
    created and documented like the Discord token Secret).
  - Traefik's default `TLSStore` uses it, so every `Ingress` gets it without a per-Ingress `tls`
    block.
  - `bring-up.sh` checks the Secret exists before installing Traefik and prints the steps if it
    doesn't.
  - Domains: ShopHub at `https://shophub.localhost`; shops at `https://<id>.shops.localhost`.
    Browsers resolve `*.localhost` to the loopback address without DNS changes.
- **Rationale**: no extra operator. The CA key never leaves the maintainer's machine, and every
  host is trusted, so `Secure` cookies just work.
- **Risk**: some browsers refuse wildcard certificates directly under `localhost`. Using a
  third-level wildcard (`*.shops.localhost`) avoids that. Checked in quickstart Scenario 1.
  Fallback: `*.shops.localtest.me` (public DNS to 127.0.0.1).
- **Alternatives**: cert-manager with a self-signed CA issuer (another release, plus an issuer
  chart, and the browser still needs a trust step); Traefik's built-in default certificate (a
  browser warning on every host).

## R3 — Shops namespace (FR-018)

- **Decision**: namespace **`shops`**, created by the `shop-operator` chart (0.4.0) as platform
  state, with label `shophub.io/shops: "true"`.
  - It holds every Shop, Wallet and DiscordChannel, and their derived resources.
  - The operator's `--watch-namespace=shops` is set from the chart value `shopsNamespace`.
  - The `shophub` chart reads the same value for its Role.
- **Rationale**: one place for all runtime state; both identities get only a namespaced Role
  there.
- **Alternatives**: namespace per owner (rejected in the spec clarifications); a separate tiny
  chart for the namespace (one more release for one object).

## R4 — Operator least privilege (FR-026, 002 R-1)

- **Decision**:
  - The manager **ClusterRole/ClusterRoleBinding is removed**. A **Role in `shops`** carries
    today's rules: shops/wallets/discordchannels (+status/finalizers), deployments, services,
    secrets, CNPG clusters, Redis, statefulsets (read), plus **ingresses** and
    **networkpolicies**.
  - A second **Role in the operator's own namespace** grants `get` on the one Discord token
    Secret by `resourceNames`. Leader election is unchanged.
  - The manager's cache is restricted to `shops` (`cache.Options.DefaultNamespaces`).
  - The Discord token is read with the uncached API reader (a namespaced `get` by name), so no
    list/watch on Secrets outside `shops` is ever needed.
  - The kubebuilder `+kubebuilder:rbac` markers switch to `namespace=shops`, and
    `hack/sync-chart.sh` writes Roles instead of the ClusterRole. The namespace is templated from
    the chart value.
  - Database kinds are still detected through discovery, which needs no RBAC beyond
    `system:discovery`.
- **Rationale**: closes R-1 exactly as the spec asks. No wildcard; nothing cluster-wide.
- **Alternatives**: keep the ClusterRole and add a ValidatingAdmissionPolicy (more machinery,
  still cluster-wide reads); RBAC by label (Kubernetes RBAC can't select by label).

## R5 — Real client address in shops (FR-022)

- **Decision**:
  - The operator injects **`TRUSTED_PROXY_CIDRS`** (chart value, default the Docker Desktop pod
    CIDR `10.244.0.0/16`, verified at task time from the node's `podCIDR`).
  - shop 0.2.0 enables ASP.NET Core `ForwardedHeaders` (`XForwardedFor | XForwardedProto`) with
    exactly those `KnownNetworks`, `ForwardLimit = 1`, and no `KnownProxies` defaults.
  - Requests from any other peer keep their peer address.
  - Every Shop also gets an owned **NetworkPolicy**: ingress to the app pods only from pods in
    namespace `traefik` and in namespace `shophub` (for the FR-016 status read), port 80.
- **Rationale**: the app can't know Traefik's pod IPs, but with the NetworkPolicy, anything in the
  pod CIDR that reaches the app is the entry point or ShopHub, both platform code.
- **Enforcement (verified 2026-11-24)**: Docker Desktop Kubernetes v1.36.1 (kind-based, CNI
  `kindnetd` v20260528) enforces NetworkPolicies. In a throwaway namespace, a deny-all policy
  blocked pod-to-pod HTTP, a `podSelector` allow rule admitted only the labelled client, and
  removing the policies restored access. The Shop policy's `namespaceSelector` form is confirmed
  live in quickstart Scenario 2.
- **Alternatives**: a shared-secret header added by a Traefik middleware (a secret in a
  non-Secret resource); trusting all addresses (forgeable from outside).

## R6 — shop 0.2.0 protections (FR-023–FR-025, FR-016)

- **Decision**:
  - **Lockout**: the existing fixed-window counter, kept per scope: one scope per client address
    (5 failures in 15 min lock that address for 15 min) and one shop-wide scope `*` (**50**
    failures in 15 min lock every address for 15 min). Both live in the shop's database
    (data-model.md) and are covered by the existing store contract suite.
  - **Rate limit**: unchanged code. The partition key is `RemoteIpAddress`, which the forwarded
    headers middleware now sets to the real client.
  - **Cookie**: `CookieSecurePolicy.Always` for the session and the anti-forgery cookies.
    Integration tests use an `https://localhost` base address.
  - **Hand-off flag**: `GET /api/shop` returns `{name, adminPasswordChanged}`. It is public: it
    only says whether the generated password has been replaced, which gives an attacker nothing
    usable.
- **Rationale**: the smallest change that closes D-1 and lets ShopHub know when to stop showing
  the generated password, with no CRD change.
- **Alternatives** (hand-off flag): an operator-maintained status field (CRD change, and the
  operator can't see the app's database); ShopHub trying the password (counts as a failed
  sign-in and can lock the shop).

## R7 — ShopHub accounts and sessions (FR-001–FR-005)

- **Decision**: reuse the `shop` patterns.
  - `PasswordHasher` (PBKDF2) and cookie authentication: 8 h, not sliding, `Secure`, HttpOnly,
    SameSite=Strict.
  - Antiforgery via the `XSRF-TOKEN` cookie and the `X-XSRF-TOKEN` header.
  - Data-Protection keys in the database.
  - A per-session id in the cookie, checked against `owner_sessions`, so sign-out ends only that
    session and a copied cookie stops working.
  - Lockout counted per (owner, client address): 5 failures in 15 min.
  - Forwarded headers trusted **only from Traefik**, stricter than shops: ShopHub resolves the
    headless Service `traefik-proxies.traefik.svc.cluster.local` (setting `TRUSTED_PROXY_HOST`)
    by DNS every 30 s and honours `X-Forwarded-For`/`-Proto` only when the TCP peer is one of
    those pod IPs, `ForwardLimit = 1`. DNS needs no Kubernetes permission, so FR-017 is
    unchanged. A failed lookup trusts nobody (fail closed: the lockout falls back to the peer
    address, which is stricter, never weaker).
  - Enforced as well by a NetworkPolicy in the `shophub` chart: the ShopHub app pods accept TCP
    8080 only from the Traefik pods (Docker Desktop enforces policies, R5).
  - Registration answers 409 "email already registered". It is unavoidable for a sign-up form,
    and it is rate-limited per client address (10 per hour).
- **Alternatives**: ASP.NET Core Identity with EF Core (an ORM and more tables than needed);
  OIDC provider (out of scope; no external IdP). For the proxy list: the pod CIDR as for shops
  (any pod could forge the header to ShopHub without a policy); reading Traefik pods through the
  Kubernetes API (needs RBAC outside `shops`, against FR-017).

## R8 — Shop record and reconciler (FR-014, FR-015)

- **Decision**:
  - The **`shops` table** is the source of truth (data-model.md).
  - API writes change the record in a transaction, then wake the reconciler. The reconciler also
    runs every 30 s.
  - For each record, it server-side applies (field manager `shophub`, `force`) the DiscordChannel,
    the current Wallet and the Shop, in that order, with labels `shophub.io/owner`,
    `shophub.io/shop` and `app.kubernetes.io/managed-by: shophub`.
  - For records in state `Deleting`, it deletes the Shop, then the Wallet(s) and the
    DiscordChannel. Once all are gone it removes the record.
  - It lists `managed-by=shophub` resources and deletes Wallets that no record references (wallet
    switch leftovers, interrupted creates).
  - Only one replica reconciles at a time, using a PostgreSQL advisory lock.
- **Idempotency**:
  - Create inserts the record with a unique `(owner_id, lower(trim(display_name)))` constraint.
    A double submit gets 409 or the existing shop.
  - Because SSA is idempotent, a crash between steps is finished on the next pass. A Wallet or
    channel can never stay without its record.
- **Rationale**: one loop gives creation, drift repair, retries and cleanup together; no
  per-request distributed transaction.
- **Alternatives**: create resources synchronously in the request (partial failures need
  compensation code); watch-based informer (more code for a 30 s target).

## R9 — Naming (FR-007, FR-009, FR-018)

- **Decision**:
  - Generated identifier: `s` + 9 random lowercase base32 characters (for example `s7k2m9qxa1`).
    It is unique in `shops`, generated on create, retried on a collision.
  - Shop `<id>` (so the host is `<id>.shops.localhost` and the admin Secret is `<id>-admin`).
  - DiscordChannel `<id>` with `channelName: <id>`.
  - Wallet `<id>-w<gen>`, where `gen` starts at 1 and increments on each address change.
- **Rationale**: valid as a DNS label and a Discord channel name; doesn't leak the display name
  or the owner into hosts or the shared Discord server.
- **Alternatives**: slug of the display name (collides across owners, and changes meaning);
  owner-prefixed names (leak the owner).

## R10 — Status mapping and the admin password hand-off (FR-008, FR-016)

- **Decision**:
  - The panel state comes from the record state and the Shop's `Ready` condition:
    - `Deleting` → "being deleted";
    - Ready=True → "ready";
    - otherwise "being prepared";
    - after 10 min not ready → "needs attention".
  - The reason codes from the 002/003 contracts map to fixed plain-language texts (contracts
    table). Unknown codes show "The platform is still working on it".
  - `POST /api/shops/{id}/admin-password` (POST, so it isn't cached or prefetched; XSRF required)
    checks ownership, reads `adminPasswordChanged` from the shop over the in-cluster Service, and
    only if false reads Secret `<id>-admin` and returns it with `Cache-Control: no-store`. The
    value is never logged.
- **Rationale**: plain language for owners; the password leaves the cluster only on an explicit
  owner action.
- **Alternatives**: showing the password in the list (it would leak into every list response).

## R11 — Testing the Kubernetes calls (FR-029)

- **Decision**: **Testcontainers k3s** (`Testcontainers.K3s`, pinned k3s image matching the
  cluster minor).
  - The CRDs come from the published `shop-operator` chart's `templates/crds.yaml` at the pinned
    version, fetched in CI and cached.
  - The suite covers: apply/status/delete, the wallet switch, idempotent create, and an
    interrupted create (fault injection in `IKubeGateway` between steps), plus the Role.
  - The Role is checked by running the gateway with a token of a ServiceAccount bound to the
    chart's Role manifest, rendered with `helm template`.
- **Rationale**: a real API server checks SSA, CRD validation (CEL immutability) and RBAC denials.
  No operator runs in the test; status is patched by the test, as envtest does.
- **Alternatives**: the in-memory fake only (misses CRD validation and RBAC); kind in CI (slower,
  harder to run locally on Windows).

## R12 — ShopHub's own database (FR-027)

- **Decision**: a CNPG `Cluster` `shophub-db` (1 instance, 1 Gi) templated in the `shophub` chart,
  in namespace `shophub`. The app gets `DATABASE_URL` from Secret `shophub-db-app` key `uri`.
  Migrations are SQL scripts with an advisory lock, as in `shop`.
- **Rationale**: platform state through the operator already on the platform; the same access
  code as `shop`.
- **Alternatives**: Redis (relational data with uniqueness constraints fits PostgreSQL better).

## R13 — Panel API and frontend (FR-005–FR-013)

- **Decision**: a REST API under `/api` (contracts/panel-api.md). The Angular panel calls the same
  origin.
  - The "open shop" link is `<a target="_blank" rel="noopener noreferrer">` (FR-010).
  - The list polls every 5 s while any shop is not ready.
  - Delete confirmation compares the typed name on the server too.
- **Alternatives**: SignalR push for status (unneeded for a 5–10 s refresh).

## R14 — Delivery and CI (FR-028, FR-029)

- **Decision**: mirror `shop`.
  - Jobs: `lint` (dotnet format, ng lint), `test` (.NET with Testcontainers PostgreSQL and k3s;
    `ng test`), and `release` on `vX.Y.Z` tags on `main` (SemVer check, refuse existing version,
    push `ghcr.io/nikola031ue/shophub:X.Y.Z`).
  - Actions pinned by SHA.
  - Chart `shophub` 0.2.0 is published by the `helm-charts` pipeline. kube-state `shophub` values
    set the image tag and hosts.
- **Alternatives**: none worth recording; consistency with `shop` matters more.
