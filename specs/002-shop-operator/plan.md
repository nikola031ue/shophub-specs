# Implementation Plan: Shop Operator (Requirements 3.1, 3.2)

**Branch**: `002-shop-operator` | **Date**: 2026-11-23 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `/specs/002-shop-operator/spec.md`

## Summary

Build a kubebuilder operator in the empty `shop-operator` repo with three namespaced kinds in
`shophub.io/v1alpha1`:

- **Shop** reconciles a placeholder Deployment (2/3 replicas), a Service, and either a CNPG
  `Cluster` or an OT `Redis`, all owned by the Shop and so garbage-collected with it. The app gets
  the wallet address and database connection through env vars and Secret references.
- **Wallet** registers a given address (EIP-55 checked) or generates a key once into an owned
  Secret, fully offline.
- **DiscordChannel** creates a Discord channel and webhook through three REST calls and stores the
  webhook in an owned Secret; a finalizer removes the channel.

All validation uses CRD schema and CEL, with no admission webhooks. Wallets and DiscordChannels
are protected by finalizers while a Shop references them.

Delivery:

- The operator image is published to GHCR on a SemVer tag.
- In helm-charts, `shop-operator` 0.2.0 carries the CRDs (in `templates/`, `keep` policy) and
  least-privilege RBAC; `shophub-discord` 0.2.0 declares the platform-alerts DiscordChannel.
- In kube-state, `cnpg` and `redis-operator` releases come from pinned upstream OCI charts. A new
  explicit allowlist in the schema check lets them through.

## Technical Context

**Language/Version**: Go (version scaffolded by kubebuilder v4, pinned in `go.mod`); Bash and
YAML for charts, kube-state and CI

**Primary Dependencies**: kubebuilder v4, controller-runtime, `go-ethereum` (`crypto`,
`common` only), Discord REST API v10 over `net/http`; upstream charts CloudNativePG 0.29.2 and
OT redis-operator 0.27.0 (research R6)

**Storage**: Kubernetes API (CRs, Secrets). Shop databases are provided by CNPG / OT
redis-operator; the operator itself stores nothing.

**Testing**: `go test` with unit tests (Discord client against `httptest`, wallet key and
checksum logic, child object builders) and envtest integration tests (reconcilers with real API
server and upstream CRDs from `test/crds/`); existing bats suites plus new allowlist cases in
kube-state; helm-charts `install` job on kind; kube-state `e2e` bring-up on kind

**Target Platform**: Docker Desktop Kubernetes (`docker-desktop`); CI `ubuntu-latest` with
envtest/kind; image `linux/amd64`

**Project Type**: Kubernetes operator (Go service) plus chart and cluster-state changes in two
existing repos

**Performance Goals**: Standard Shop Ready < 5 min on the local cluster (SC-001); scale change
< 2 min (SC-002); cleanup < 2 min (SC-003); bring-up < 15 min (SC-006)

**Constraints**:

- No admission webhooks.
- No blockchain RPC (FR-011) and no real Discord in CI.
- No secrets in git, status or logs (SC-007).
- The operator must start without CNPG/Redis or a Discord Secret (R5, R15).
- The 5.3 layout is immutable.
- Solo developer, about 25 tasks.

**Scale/Scope**: 3 CRDs, 3 reconcilers, 1 small Discord client; tens of Shops on one local
cluster; 3 repos touched (`shop-operator` new code, `helm-charts`, `kube-state`)

All items resolved in [research.md](research.md); no NEEDS CLARIFICATION remain.

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.* Constitution v2.1.0.

| Principle | Check | Pre-research | Post-design |
|-----------|-------|--------------|-------------|
| I. Immutable IaC layout | `charts/shop-operator` and `charts/shophub-discord` keep `Chart.yaml`/`values.yaml`/`templates/`. New kube-state folders `cnpg/` and `redis-operator/` follow the 5.3 release-folder pattern (`helm.yaml` + `values.yaml`), like `shophub-discord` in 001. The allowlist lives in hidden `.github/`. Upstream charts are not copied into `helm-charts`, which 5.3 reserves for charts "you write yourself". | PASS | PASS |
| II. Repo-per-service, trunk-based | Operator in its own repo `shop-operator` (GitHub `shop-operator-spec`), PR-only `main` with branch protection applied **before the first merge**, squash merges, Conventional Commits, image `ghcr.io/nikola031ue/shop-operator:X.Y.Z` from a `vX.Y.Z` tag, no `latest` (placeholder Shop image `nginx:1.27.3`) | PASS | PASS |
| III. Tested before merge | Unit plus envtest integration tests required in `shop-operator` CI; every bug fix gets a test; chart `install` and kube-state `e2e` stay required. Testcontainers N/A (no .NET). | PASS | PASS |
| IV. Sepolia, tested locally | Wallet is offline only: key generation and address validation, no RPC. CI never contacts any chain, so Anvil isn't needed (spec Q1). Generated keys hold testnet funds only. | PASS | PASS |
| V. Platform in git, shops via API | Operator, CRDs, RBAC, `cnpg`, `redis-operator` and the platform DiscordChannel are platform state in charts and kube-state. Shop/Wallet/DiscordChannel are user resources the operator never creates. Databases, workloads and Secrets are derived resources the operator creates as Shop-owned children (v2.1.0). The ClusterRole is justified below. Exceptions: the hand-applied test CRs and the Discord token bootstrap, both in Complexity Tracking. | PASS (with tracked items) | PASS (with tracked items) |
| VI. Secrets in secret stores | Private keys, webhook URLs, the bot token and DB credentials live only in K8s Secrets and are referenced by name; chart values hold no secrets; tests assert status and logs carry no secret values (SC-007); CI uses `GITHUB_TOKEN` only | PASS | PASS |
| VII. Per-Shop observability | Out of scope for this feature (spec Assumptions). This feature provides what VII needs later: per-Shop DiscordChannel, `shophub.io/owner` and `shophub.io/shop` labels on every child, and the platform-alerts channel. Dashboards and alert rules come in the observability feature. | PASS (deferred, not violated) | PASS |

Gate result: **PASS** — no unjustified violations.

## Project Structure

### Documentation (this feature)

```text
specs/002-shop-operator/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── crds.md
│   └── charts-and-kube-state.md
├── checklists/requirements.md
└── tasks.md             # /speckit-tasks
```

### Source Code

```text
shop-operator/                     # new code (kubebuilder v4 layout)
├── api/v1alpha1/                  # shop_types.go, wallet_types.go, discordchannel_types.go (+CEL markers)
├── internal/
│   ├── controller/                # shop_, wallet_, discordchannel_controller.go + *_test.go (envtest)
│   ├── discord/                   # client.go (3 endpoints, 429 handling) + client_test.go (httptest)
│   ├── wallet/                    # keygen.go, checksum.go + tests
│   └── resources/                 # builders: deployment, service, cnpg cluster, redis (unstructured) + tests
├── cmd/main.go                    # flags: --shop-image(-tag), --discord-guild-id, --discord-token-secret
├── config/                        # generated CRDs and RBAC (source for the chart)
├── test/crds/                     # upstream CNPG + Redis CRDs at pinned versions (envtest)
├── hack/fetch-crds.sh, hack/sync-chart.sh
├── Dockerfile
└── .github/workflows/ci.yaml      # lint, test (PR); release (tag vX.Y.Z → GHCR)

helm-charts/
├── charts/shop-operator/          # 0.2.0: templates/{crds,deployment,serviceaccount,rbac}.yaml, values.yaml
├── charts/shophub-discord/        # 0.2.0: templates/discordchannel.yaml
└── .github/workflows/charts.yaml  # install job: shop-operator first

kube-state/
├── clusters/local/cluster.yaml    # releaseOrder adds cnpg, redis-operator
├── clusters/local/cnpg/           # helm.yaml, values.yaml (new)
├── clusters/local/redis-operator/ # helm.yaml, values.yaml (new)
├── clusters/local/shop-operator/  # tag 0.2.0, dependsOn, values (discord.guildID)
├── clusters/local/shophub-discord/ # tag 0.2.0
└── .github/
    ├── upstream-charts.yaml       # allowlist (new)
    ├── scripts/check-schema.sh    # allowlist rule
    └── tests/check-schema.bats    # allowlist cases
```

**Structure Decision**: Three repositories, matching Principle II. All new Go code is in
`shop-operator`; `helm-charts` and `kube-state` get changes only, as in the contract. Merge order:
`shop-operator` (image 0.1.0 published and Public) → `helm-charts` (charts 0.2.0 published and
Public) → `kube-state`.

## Delivery Notes (solo, ~25 tasks)

- Suggested story order: Wallet and DiscordChannel types and controllers before Shop, because
  Shop readiness depends on both. That is still one MVP slice per spec story: US1 (Shop + Wallet,
  with DiscordChannel against the fake), US2 (charts/kube-state), US3 (real Discord check).
- `shop-operator` needs branch protection (contexts `lint`, `test`) before its first merge,
  using the same command shape as the 001 READMEs.
- Risk: CNPG image pulls on first bring-up can take minutes; SC-001 is measured with images
  already cached, SC-006 covers first install.
- Risk: OT Redis status fields differ between versions, so readiness is checked on the
  StatefulSet instead (R12).

## Complexity Tracking

| Violation / exception | Why Needed | Simpler Alternative Rejected Because |
|-----------------------|------------|--------------------------------------|
| ClusterRole for the operator (Principle V allows it only if justified) | Shops live in user namespaces chosen at runtime by the future backend; no fixed namespace list exists at install time | Namespace-scoped Roles can't be created for namespaces that don't exist yet. Mitigation: exact resources and verbs, no wildcards, Secret cache limited by label (R14) |
| One-time `kubectl create secret` for the Discord bot token (Principle V "no raw kubectl create") | The token can't be in git (Principle VI), and no secret manager is in scope | Having `bring-up.sh` create it changes the 001 contract (never applies manifests). Sealed Secrets/SOPS add a platform component. The operator works without it (`DiscordCredentialsMissing`) (R15) |
| Maintainer hand-applies test Shop/Wallet/DiscordChannel CRs in a throwaway namespace (quickstart 3–5) | The ShopHub backend, the only allowed creator of user resources, doesn't exist yet | Waiting for the backend would leave US1 unverifiable on a real cluster. Limited to namespace `qs`, deleted after validation; automated tests use envtest instead |
