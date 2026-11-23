# Research: Shop Operator

All Technical Context unknowns are resolved below. Versions marked *(verified)* were checked
live on 2026-11-23 with `helm show chart` against the public registries.

## R1 — Operator framework and language

- **Decision**: Go + kubebuilder v4 (standard layout `api/`, `internal/controller/`, `cmd/`,
  `config/`), controller-runtime as scaffolded. The Go and controller-runtime versions are the
  ones kubebuilder scaffolds at T001 and are pinned in `go.mod`.
- **Rationale**: Mandated by the constitution (Technology constraints, Principle III envtest).
- **Alternatives**: Operator SDK (adds OLM tooling we don't need); a Helm-based operator (cannot
  generate keys or call Discord).

## R2 — API group and version

- **Decision**: `shophub.io/v1alpha1`; kinds `Shop`, `Wallet`, `DiscordChannel`, all
  namespaced. Short names `shop`, `wallet`, `dch`.
- **Rationale**: `v1alpha1` signals an unstable API while the backend and Shop app don't exist.
  Group is a name only; nothing is served from it.
- **Alternatives**: `v1` now (premature); group under `nikola031ue.github.io` (long, no benefit).

## R3 — Validation without admission webhooks

- **Decision**: Use CRD OpenAPI validation and CEL rules only: enums for `availability` and
  `database`, required refs, `^0x[0-9a-fA-F]{40}$` for a Wallet address,
  `^[a-z0-9-]{1,100}$` for Discord channel names, and `self == oldSelf` for immutable fields
  (`shopName`, `database`, `wallet.spec.address`). The EIP-55 checksum of a mixed-case address
  can't be computed in CEL, so the controller checks it and reports `InvalidChecksum` in status.
- **Rationale**: Webhooks need TLS certificates (cert-manager or self-managed); CEL covers every
  rejection the spec requires (SC-004) at no operational cost.
- **Alternatives**: Validating webhook (rejected: certificates, extra failure mode).

## R4 — Database resources: typed vs. unstructured

- **Decision**: Create CNPG `Cluster` (`postgresql.cnpg.io/v1`) and OT `Redis`
  (`redis.redis.opstreelabs.in/v1beta2`) as `unstructured.Unstructured` objects built from small
  templates; envtest loads the upstream CRDs from `test/crds/`, fetched at the pinned chart
  versions by `hack/fetch-crds.sh`.
- **Rationale**: Importing the upstream Go API modules pulls large dependency trees and couples
  our build to their release cadence; we only need a handful of fields.
- **Alternatives**: Typed upstream APIs (heavier, version coupling).

## R5 — Missing database operators at runtime

- **Decision**: At startup the operator checks via discovery whether each database kind is
  served. It only registers an `Owns()` watch for kinds that exist. Shops needing a missing kind
  report `Ready=False, reason=DatabaseOperatorMissing`. The operator never crashes for a missing
  optional CRD.
- **Rationale**: controller-runtime fails to start if it watches an unknown kind; the helm-charts
  `install` CI job installs the chart on a bare kind cluster without CNPG/Redis.
- **Alternatives**: Hard dependency (breaks chart CI); unconditional watch (crash loop).

## R6 — Upstream database operators (FR-024)

- **Decision**:
  - CloudNativePG chart `oci://ghcr.io/cloudnative-pg/charts/cloudnative-pg` **0.29.2**
    (operator 1.30.2) *(verified)*; CRDs ship in `templates/`.
  - OT-Container-Kit redis-operator chart `oci://ghcr.io/ot-container-kit/charts/redis-operator`
    **0.27.0** *(verified)*; CRDs ship in `crds/`; watches all namespaces by default; webhook off.
  - New `kube-state` releases `cnpg` (namespace `cnpg-system`) and `redis-operator`
    (namespace `redis-operator`), both before `shop-operator` in `releaseOrder`.
- **Rationale**: Both publish anonymous-pullable OCI charts, so `bring-up.sh` and the
  `charts-exist` CI job work unchanged. Requirement 1.2 names CNPG for PostgreSQL; OT redis-operator
  is the common open-source Redis operator (REDB/Redis Enterprise needs a licence).
- **Alternatives**: Spotahome redis-operator (unmaintained); Redis Enterprise operator (licence);
  `ghcr.io/ot-container-kit/helm-charts/...` (does not exist, `403`).
- **Note**: bring-up's CRD wait (feature 001 FR-023) already reads CRDs from both
  `helm show crds` and `helm get manifest`, so CNPG's template-shipped CRDs are waited on.

## R7 — kube-state upstream chart allowlist

- **Decision**: New file `kube-state/.github/upstream-charts.yaml` listing allowed upstream charts
  as exact `{url, version}` pairs. `check-schema.sh` accepts a `helm.yaml` URL if it starts with
  `$OCI_BASE/` **or** exactly matches an allowlisted `url` with a tag equal to that entry's
  `version`. Anything else fails with a message naming the allowlist. Bumping an upstream chart
  therefore needs a reviewed edit to both the release and the allowlist.
- **Rationale**: Matches the user's request for an explicit, pinned allowlist; the file is under
  hidden `.github/` and leaves the 5.3 layout untouched (Principle I).
- **Alternatives**: Allow any `oci://` URL (no supply-chain control); mirror the charts into our
  GHCR (extra publishing pipeline).

## R8 — CRDs in the shop-operator chart

- **Decision**: CRDs go in `charts/shop-operator/templates/crds.yaml`, annotated
  `helm.sh/resource-policy: keep`, generated from kubebuilder `config/crd` by
  `shop-operator/hack/sync-chart.sh` (also syncs the ClusterRole rules). The synced files are
  committed to `helm-charts` through a normal PR.
- **Rationale**: Helm never upgrades files in `crds/`, so US2 scenario 2 (upgrade) would silently
  keep old CRDs. `keep` prevents uninstall from deleting every Shop.
- **Alternatives**: `crds/` folder (no upgrades); separate CRD chart (extra release, layout).

## R9 — Discord integration

- **Decision**: A small client over Discord REST API v10 with `net/http`:
  `POST /guilds/{guild}/channels` (type 0, topic contains `shophub-uid:<CR uid>`),
  `POST /channels/{id}/webhooks`, `DELETE /channels/{id}`, `GET /guilds/{guild}/channels`.
  It authenticates with `Authorization: Bot <token>` and honours `429 retry_after`. The bot token
  comes from a Secret (`discord.tokenSecretName`, key `token`) and the guild ID from chart values.
  Idempotency: `status.channelID`; if it is empty, look up a channel whose topic carries the CR
  UID before creating a new one. The webhook URL goes into Secret `<name>-discord-webhook`
  (key `url`), owned by the DiscordChannel. A finalizer deletes the Discord channel.
  Tests use an `httptest` fake Discord behind a `Client` interface.
- **Rationale**: Three endpoints don't justify a Discord SDK dependency; the fake keeps CI
  offline.
- **Alternatives**: `discordgo` library (large surface, gateway code we don't use).

## R10 — Wallet key generation (FR-009–FR-011)

- **Decision**: Use `go-ethereum` `crypto.GenerateKey` and `crypto.PubkeyToAddress` for new keys,
  and `common.IsHexAddress` plus an EIP-55 comparison for checksums. Generated keys go into Secret
  `<name>-wallet-key` (key `privateKey`, hex), owned by the Wallet. If that Secret exists, the
  address is always derived from it and never regenerated. If the Secret is gone but
  `status.address` is already set, the key is lost: the Wallet reports `KeyLost`, keeps the old
  address and generates nothing, because a silent new key would change the payment address.
  No RPC client is created.
- **Rationale**: Standard, audited implementation; offline only, so Principle IV is satisfied
  without Anvil (spec clarification Q1).
- **Alternatives**: `decred/secp256k1` + `x/crypto/sha3` (fewer deps but hand-rolled address
  derivation).

## R11 — Shop workload and exposure (FR-002, FR-007, FR-008)

- **Decision**:
  - A Deployment `<shop>` (replicas 2/3) and a ClusterIP Service `<shop>` on port 80. The image
    comes from operator flags `--shop-image` and `--shop-image-tag`, which the chart sets from
    `shopImage.repository/tag` (default `nginx` `1.27.3`).
  - Env: `SHOP_NAME`, `WALLET_ADDRESS`; for PostgreSQL `DATABASE_URL` from CNPG's generated
    Secret `<shop>-db-app` (key `uri`); for Redis `REDIS_URL=redis://<shop>-redis.<ns>.svc:6379`.
  - A changed wallet changes the pod template, so Kubernetes rolls the app automatically.
  - `status.url` is `http://<shop>.<ns>.svc.cluster.local`.
- **Rationale**: Docker Desktop ships no ingress controller. An in-cluster address satisfies
  FR-008 for the ShopHub backend; browser exposure (Ingress) comes with the ShopHub/Shop features
  that add an ingress controller.
- **Alternatives**: Ingress now (needs a controller release, hostnames: out of minimal scope);
  NodePort (port collisions per shop).
- **Note**: Redis runs without a password (OT standalone default). Acceptable on the local
  cluster; feature 3 adds `redisSecret` when the Shop app uses Redis for real.

## R12 — Readiness and status (FR-017)

- **Decision**: Each kind gets `status.conditions` (`Ready` with `reason`/`message`,
  `lastTransitionTime`) and `observedGeneration`. A Shop is Ready when all of these hold:
  - the Wallet and DiscordChannel are Ready;
  - the database is ready (CNPG condition `Ready=True`; Redis StatefulSet `<shop>-redis` has
    `readyReplicas ≥ 1`);
  - Deployment `availableReplicas` equals the desired count.

  Reason codes are fixed in [contracts/crds.md](contracts/crds.md). Requeue with
  controller-runtime's exponential backoff on errors; 30 s periodic requeue while not Ready.
- **Rationale**: Standard condition semantics; `kubectl get` printer columns show them.

## R13 — Reference protection (FR-010, FR-016)

- **Decision**: Field indexes on `Shop.spec.walletRef.name` and `Shop.spec.discordChannelRef.name`.
  Wallet and DiscordChannel get a finalizer that blocks removal while any Shop in the namespace
  references them. Status `usedBy` lists them, and the reason is `InUse` while deletion waits.
  Shops watch Wallets and DiscordChannels through these indexes, so a Shop re-reconciles when its
  referenced resource changes (for example a wallet rotation).
- **Rationale**: Native Kubernetes pattern; no webhook needed.

## R14 — RBAC scope

- **Decision**: One ClusterRole bound to the operator's ServiceAccount. Shops live in user
  namespaces chosen by the future backend, so namespace-scoped Roles can't be known at install
  time. Rules list exact resources and verbs:
  - shophub.io kinds (and their status and finalizers)
  - deployments, services, secrets, events
  - `apps/statefulsets` — `get`, `list`, `watch` only (Redis readiness, R12)
  - `clusters.postgresql.cnpg.io`, `redis.redis.opstreelabs.in`
  - leases (leader election, own namespace)

  No wildcards. Rules come from kubebuilder `+kubebuilder:rbac` markers on the reconcilers,
  because envtest doesn't enforce RBAC and a missing rule only shows on a real cluster. The
  secrets cache is limited to objects labelled `app.kubernetes.io/managed-by=shop-operator`. The
  hand-created, unlabelled Discord token Secret is therefore read with the uncached API reader
  (`mgr.GetAPIReader()`), never through the cached client.
- **Rationale**: Principle V allows a ClusterRole when justified; it is justified here and
  recorded in plan.md Complexity Tracking.

## R15 — The Discord bot token

- **Decision**: The maintainer creates Secret `shop-operator-discord` (key `token`) in
  `shop-operator-system` once per cluster, with a documented `kubectl create secret` command.
  Without it the operator runs normally and DiscordChannels report
  `DiscordCredentialsMissing`.
- **Rationale**: Principle VI forbids the token in git; no secret manager is in scope. This
  single bootstrap step is the only exception to "no raw kubectl create" and is recorded in
  Complexity Tracking.
- **Alternatives**:
  - `bring-up.sh` creates it from an env var: changes the feature 001 contract (FR-022, never
    kubectl apply).
  - Sealed Secrets or SOPS: a new platform component, out of minimal scope.

## R16 — Delivery and CI

- **Decision**:
  - `shop-operator` repo workflow `ci.yaml`: on PR, `lint` (`go vet`, golangci-lint) and `test`
    (unit + envtest via `setup-envtest`, CRDs from `test/crds/`). On tag `vX.Y.Z`, `release`
    builds a linux/amd64 image and pushes `ghcr.io/nikola031ue/shop-operator:X.Y.Z` with
    `GITHUB_TOKEN`. The package is made Public once, like the charts.
  - Chart versions: `shop-operator` 0.1.0 → **0.2.0** (appVersion `0.1.0`), `shophub-discord`
    0.1.0 → **0.2.0**.
  - helm-charts `install` job: install `shop-operator` first, then the rest.
- **Rationale**: Principles II and III. The image must exist and be public before the
  helm-charts PR, because the `install` job pulls it anonymously.
- **Order**: shop-operator image `0.1.0` → helm-charts PR (charts 0.2.0) → kube-state PR
  (allowlist, `cnpg`, `redis-operator`, bumped versions).
