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
  - A second **Role in the operator's own namespace** (`…-platform-channels`) covers
    DiscordChannels (with status and finalizers) and Secrets there. Leader election is unchanged.
  - **Platform alerts channel (decided 2026-11-24 during 1B)**: the `platform-alerts`
    DiscordChannel from the `shophub-discord` chart lived in `shophub`. Watching only `shops`
    would have left it unreconciled: a lost webhook Secret would never come back, and a delete
    would hang on its finalizer. It moves to the operator's namespace (`shop-operator-system`),
    which the operator already needs. Rejected: moving it into `shops`, where ShopHub's Role
    could change or delete it; watching `shophub` too, which would give the operator Secret
    access next to ShopHub's database credentials.
  - The manager's cache: every kind in `shops` only (`cache.Options.DefaultNamespaces`);
    DiscordChannels and operator-labelled Secrets also in the operator's namespace
    (`ByObject.Namespaces`). Shops and Wallets there are not cached or reconciled.
  - The Discord token is still read with the uncached API reader.
  - The manager markers stay as they are, and `hack/sync-chart.sh` writes their rules into a
    Role in `.Values.shopsNamespace`. The own-namespace rules come from markers with
    `namespace=system`; the script writes them into the release namespace.
  - Database kinds are still detected through discovery, which needs no RBAC beyond
    `system:discovery`.
- **Rationale**: closes R-1 exactly as the spec asks. No wildcard; nothing cluster-wide.
- **Alternatives**: keep the ClusterRole and add a ValidatingAdmissionPolicy (more machinery,
  still cluster-wide reads); RBAC by label (Kubernetes RBAC can't select by label).

## R5 — Real client address in shops (FR-022)

- **Decision**:
  - **Revised 2026-11-25 (before the 0.3.0 release)**: forwarded headers are trusted **only from
    Traefik**, the same rule as ShopHub (R7). The operator injects **`TRUSTED_PROXY_HOST`**
    (`--trusted-proxy-host`, chart value `trustedProxyHost`, default
    `traefik-proxies.traefik.svc.cluster.local`, the headless Service from 1A). shop 0.2.0
    resolves it by DNS every 30 s and honours `X-Forwarded-For`/`-Proto` only when the TCP peer
    is one of those addresses, `ForwardLimit = 1`. Unset or a failed lookup trusts nobody (fail
    closed: the peer address is used, which only makes the lockout stricter).
  - Requests from any other peer keep their peer address.
  - Every Shop also gets an owned **NetworkPolicy**: ingress to the app pods only from pods in
    namespace `traefik` and from the ShopHub app pods (namespace `shophub` and the chart's pod
    labels `app.kubernetes.io/name=shophub,app.kubernetes.io/instance=shophub` in one peer; operator 0.3.1), port 80. Other pods in `shophub`, such as its
    database, are not admitted. ShopHub is admitted for one
    concrete feature, the FR-016 hand-off: ShopHub reads `GET /api/shop` over the in-cluster
    Service. It is an untrusted peer: its forwarded headers are ignored.
- **Rationale**: the first design trusted the whole pod CIDR (`TRUSTED_PROXY_CIDRS=10.244.0.0/16`)
  and relied on the NetworkPolicy. But the policy admits `shophub`, whose pods are in that CIDR,
  so a ShopHub pod (or anything running in that namespace) could set `X-Forwarded-For` to a new
  address on every attempt. With the lockout keyed on (account, address), that is unlimited
  guessing, bounded only by the shop-wide ceiling. Trusting the Traefik pod addresses by DNS
  closes it without Kubernetes permissions or a shared secret.
- **Enforcement (verified 2026-11-24)**: Docker Desktop Kubernetes v1.36.1 (kind-based, CNI
  `kindnetd` v20260528) enforces NetworkPolicies. In a throwaway namespace, a deny-all policy
  blocked pod-to-pod HTTP, a `podSelector` allow rule admitted only the labelled client, and
  removing the policies restored access. The Shop policy's `namespaceSelector` form is confirmed
  live in quickstart Scenario 2.
- **Observed client address at Traefik (verified 2026-11-24)**: a throwaway `traefik/whoami`
  behind an IngressRoute on `shophub.localhost`, called with `curl https://shophub.localhost/…`
  from Windows, reported `X-Forwarded-For: 172.18.0.5` / `X-Real-Ip: 172.18.0.5` (peer
  `10.244.0.64` = the Traefik pod).
  - `172.18.0.5` is the Docker Desktop LoadBalancer proxy container (`kindccm-…`, the Service's
    EXTERNAL-IP) on the `kind` Docker network. It is neither a pod (pod CIDR `10.244.0.0/24`) nor a
    node (`desktop-control-plane` is `172.18.0.2`), so the check passes.
  - **Limitation**: that proxy opens a new connection to Traefik, so locally Traefik sees the
    proxy's address, not the Windows client's. `externalTrafficPolicy: Local` doesn't change this.
    Every local client therefore shares one address. Rate limits (orders, registration) stay per
    client address and so act as one shared bucket on Docker Desktop.
  - **Why this is acceptable**: lockouts are keyed by **(account, client address)**, not by the
    address alone (shop R6, ShopHub R7). Failures against one account from the shared address
    don't lock any other account, so one local attacker can't lock out every owner. The shared
    rate-limit bucket only matters with many local clients, and the maintainer is the only one.
    On a cluster whose load balancer keeps the source address, the same code keys on the real
    client. The
    trust chain itself (only Traefik's `X-Forwarded-For` is honoured, forged headers are ignored)
    is unaffected.
- **Alternatives**: a shared-secret header added by a Traefik middleware (a secret in a
  non-Secret resource); trusting all addresses (forgeable from outside).

## R6 — shop 0.2.0 protections (FR-023–FR-025, FR-016)

- **Decision**:
  - **Lockout**: the existing fixed-window counter, kept per scope: one scope per **(account,
    client address)** (5 failures in 15 min lock that account from that address for 15 min) and
    one shop-wide scope `*` (**50** failures in 15 min lock every address for 15 min). The shop
    has one admin account (`admin`), but the key carries the account so a second account would
    never share a counter, matching ShopHub (R7), and the shared local address (R5) stays harmless. Both live in the shop's database
    (data-model.md) and are covered by the existing store contract suite.
  - **Rate limit**: unchanged code, still per client address only. The partition key is
    `RemoteIpAddress`, which the forwarded
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

- **Decision**: reuse the `shop` patterns. Since 2026-11-25 (Phase 2) the code itself is ported
  from shop 0.2.0 with its tests: trusted-proxy resolver, lockout store, session check, cookie
  policy, rate limiting.
  - `PasswordHasher` (PBKDF2) and cookie authentication: 8 h, not sliding, `Secure`, HttpOnly,
    SameSite=Strict.
  - Antiforgery via the `XSRF-TOKEN` cookie and the `X-XSRF-TOKEN` header.
  - Data-Protection keys in the database.
  - A per-session id in the cookie, checked against `owner_sessions`, so sign-out ends only that
    session and a copied cookie stops working.
  - Lockout counted per (owner, client address): 5 failures in 15 min, as fixed-window counters
    in `signin_lockouts` (shop 0.2.0's store). Plus a per-account ceiling: 50 failures for one
    owner across addresses in 15 min (maintainer's choice, 2026-11-25). Rejected: no ceiling
    (a botnet gets 5 guesses per address per 15 min against one account) and a global ceiling
    as in the shop (one attacker could lock out every owner). Accepted trade-off: someone with
    10 or more addresses can lock one victim's account for 15 min. Attempts while either scope
    is locked count nowhere (shop 0.2.0 burst fix).
  - Sessions: the shop checks one admin session version on every request; ShopHub checks a
    per-session row (`owner_sessions`) instead. Sign-out deletes that row; a password change
    deletes every other row of the owner (FR-002).
  - Forwarded headers trusted **only from Traefik**, stricter than shops: ShopHub resolves the
    headless Service `traefik-proxies.traefik.svc.cluster.local` (setting `TRUSTED_PROXY_HOST`)
    by DNS every 30 s and honours `X-Forwarded-For`/`-Proto` only when the TCP peer is one of
    those pod IPs, `ForwardLimit = 1`. DNS needs no Kubernetes permission, so FR-017 is
    unchanged. A failed lookup trusts nobody (fail closed: the lockout falls back to the peer
    address, which is stricter, never weaker).
  - Enforced as well by a NetworkPolicy in the `shophub` chart: the ShopHub app pods accept TCP
    8080 only from the Traefik pods (Docker Desktop enforces policies, R5).
  - No account enumeration (FR-001, decided 2026-11-25; replaces the earlier 409 "email already
    registered"): registration answers 202 with no body and no cookie whether or not the email
    exists, and always computes one password hash; the panel then signs in. Sign-in with an
    unknown email returns the wrong-password 401 and verifies against a fixed dummy hash, so both
    paths cost one PBKDF2 run. Registration is rate-limited per client address (10 per hour).
    Residual, accepted: without email verification (out of scope), registering an email and then
    signing in with that password tells whether the email was taken (sign-in fails). That costs a
    registration (10/hour/address) and a sign-in failure (lockout), and works only one email at a
    time; email verification would close it.
- **Alternatives**: ASP.NET Core Identity with EF Core (an ORM and more tables than needed);
  OIDC provider (out of scope; no external IdP). For the proxy list: the pod CIDR (any pod the
  policy admits could forge the header; shops dropped it too, R5); reading Traefik pods through the
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
    DiscordChannel, each by name from the record. Once all are gone it removes the record.
  - It never lists or selects cluster objects. Old Wallet generations are deleted by name
    (`<id>-w1` up to the one before the current) after a wallet switch. An object without a
    record, even with ShopHub's labels, is never touched (FR-015).
  - An interrupted create can't leave an orphan: the record exists before any object, and the
    next pass applies the rest.
  - Only one replica reconciles at a time. A pass holds a transaction-level PostgreSQL advisory
    lock (`pg_try_advisory_xact_lock`), which is released on commit, rollback or a lost
    connection; a replica that can't get it skips the pass.
- **Idempotency**:
  - Create inserts the record with a unique `(owner_id, lower(trim(display_name)))` constraint,
    under a row lock on the owner. A double submit or retry with the same name and settings gets
    the existing shop; the same name with other settings gets 409 (data-model.md `shops`).
  - Because SSA is idempotent, a crash between steps is finished on the next pass, and a pass
    with nothing to change leaves every `resourceVersion` as it was. A Wallet or channel can
    never stay without its record.
- **Rationale**: one loop gives creation, drift repair, retries and cleanup together; no
  per-request distributed transaction.
- **Alternatives**: create resources synchronously in the request (partial failures need
  compensation code); watch-based informer (more code for a 30 s target).
- **Superseded**: an earlier draft listed `managed-by=shophub` objects and pruned Wallets
  without a record. That was dropped, because ShopHub selects nothing by label (T014): a pruning
  mistake could reach objects ShopHub doesn't own.

## R9 — Naming (FR-007, FR-009, FR-018)

- **Decision**:
  - Generated identifier: `s` + 9 random lowercase base32 characters (for example `s7k2m4qxab`).
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

## R15 — Wallet address checksum in ShopHub (FR-006, FR-009)

- **Decision**: ShopHub checks a wallet address before it creates anything (FR-006), with the
  same rule as shop-operator's `wallet.NormalizeAddress`:
  - `0x` + 40 hex digits;
  - all-lowercase or all-uppercase carries no checksum and is accepted;
  - mixed case must match EIP-55 exactly.

  The address is stored and sent to the Wallet in EIP-55 form. The code is
  `shophub/src/ShopHub.Api/Shops/Eip55.cs`.
- **Keccak-256 is written by hand** (about 100 lines, in `Eip55.cs`):
  - EIP-55 needs Ethereum's Keccak-256: the original Keccak with padding byte `0x01`.
  - .NET's `SHA3_256` is the FIPS 202 SHA3-256, which uses padding byte `0x06`, so it gives
    different digests and can't be used.
  - .NET has no Keccak of its own, and a crypto package (Nethereum, BouncyCastle) would add a
    large dependency for one function.
- **Scope**: the hash is used only to compute and check address checksums. It is never used
  for passwords, tokens, signatures or anything else security-related. Passwords use
  `PasswordHasher` (PBKDF2), R7.
- **Test vectors** (`tests/ShopHub.Api.Tests/Unit/ShopRulesTests.cs`):
  - Keccak-256 of the empty input is
    `c5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`.
  - Keccak-256 of `abc` is
    `4e03657aea45a94fc7d47ba826c8d667c0d1e6e33a64a036ec44f58fa12d6c45`.
  - **Cross-check against SHA3:** the same sponge with padding byte `0x06` must equal the
    platform's `SHA3_256` at input lengths 0, 1, 135, 136, 137, 271, 272 and 500 bytes. 136
    bytes is the rate, so these lengths cross the block boundaries.
  - **EIP-55 examples** must pass unchanged: `0x5aAeb6053F3E94C9b9A09f33669435E7Ef1BeAed`,
    `0xfB6916095ca1df60bB79Ce92cE3Ea74c37c5d359`,
    `0xdbF03B407c01E7cD3CBea99509d93f8DDDC8C6FB` and
    `0xD1220A0cf47c7B9Be7A2E6BA89F429762e7b9aDb`. These are the vectors shop-operator's wallet
    tests use.
  - **Normalising and refusing:** an all-lowercase or all-uppercase address is converted to
    its EIP-55 form. A single wrong letter case is refused as a checksum error.
- **Alternatives**:
  - Nethereum or BouncyCastle: a large dependency for one function.
  - No checksum check in ShopHub, leaving it to the operator: a typo would then surface only
    after the resources exist, as `InvalidChecksum`, instead of as a field error before
    anything is created.
