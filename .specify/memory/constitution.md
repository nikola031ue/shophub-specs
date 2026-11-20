<!--
Sync Impact Report
==================
Version change: 1.0.0 → 2.0.0
Bump rationale: MAJOR — Principle III is redefined (test-first and NON-NEGOTIABLE status
removed) and Principle V is redefined (kube-state no longer holds all cluster state). Both
are backward-incompatible changes to existing principles.

Modified principles:
  - III. Test-First (NON-NEGOTIABLE) → III. Tested Before Merge
      Unit + integration tests stay mandatory before merge; test-first is now recommended
      (SHOULD), not required; NON-NEGOTIABLE marker removed.
  - V. Declarative Deployment via Operators & Helm → V. Declarative Deployment: Platform in
    Git, Shops via API
      kube-state holds platform state only; user-created Shop, Wallet, DiscordChannel and
      database CRs are runtime state created by the ShopHub backend through the Kubernetes
      API under a least-privilege ServiceAccount + RBAC, and are not stored in git.
Modified sections:
  - Development Workflow & Quality Gates: solo development reclassified from "compliance" to
    a recorded deviation from Requirement 5.1 (D-1), mitigated by self-review and required CI;
    Definition of done no longer demands tests written first.
  - Governance: only Principle I remains NON-NEGOTIABLE; project-level deviations are
    recorded in this constitution.
Added sections: none (new "Recorded deviations" list lives inside the Workflow section)
Removed sections: none

Follow-up TODOs: none
-->

# ShopHub Constitution

## Core Principles

### I. Immutable IaC Layout (Requirement 5.3) (NON-NEGOTIABLE)

The folder structures of the `helm-charts` and `kube-state` repositories defined in
Requirement 5.3 of `docs/ShopHub.md` are fixed and MUST NOT be changed.

- `helm-charts` MUST contain a root `README.md` and a `charts/` directory. Each chart sits in
  `charts/<name>/` with `Chart.yaml`, `values.yaml` and `templates/` (and `charts/` for
  subcharts where needed, as in `shophub`).
- `kube-state` MUST contain a root `README.md` and `clusters/<cluster>/` (at minimum
  `clusters/local/`) holding `cluster.yaml` plus one folder per release (`shop-operator/`,
  `shophub/`, `shophub-discord/`, …), each with `helm.yaml` (chart reference in OCI format)
  and `values.yaml` (value overrides).
- New charts or releases MAY be added only by following the same pattern. Renaming, moving,
  nesting differently or adding parallel top-level layouts is forbidden.
- Adopting a GitOps tool (Argo CD / Flux) MUST NOT alter this layout.

Rationale: Requirement 5.3 is eliminatory; a project that breaks it is not reviewed at all.

### II. Repository-per-Service & Trunk-Based Delivery

- Every deployable service (shophub, shop, shop-operator) MUST live in its own Git repository;
  IaC lives in `helm-charts` and `kube-state`; specs live in `shophub-specs`.
- Development MUST be trunk-based: short-lived branches off `main`, merged back via Pull
  Request. Direct pushes to `main` are forbidden (branch protection enforced).
- A PR MUST NOT be mergeable unless its CI pipeline passes (required status checks).
- `main` MUST keep a linear history (squash or rebase merges only; no merge commits).
- Every commit message MUST follow Conventional Commits (`feat:`, `fix:`, `chore:`, …).
- Container images MUST be tagged with Semantic Versioning (`MAJOR.MINOR.PATCH`) and published
  to a container registry; `latest` MUST NOT be referenced by any chart or kube-state values.

Rationale: implements Requirements 5.1 and 5.2 and keeps each service independently releasable.

### III. Tested Before Merge

- Each service MUST have unit tests and integration tests covering the change, and both MUST
  pass in CI before a PR can be merged.
- Writing tests first (Red → Green → Refactor) is RECOMMENDED but not required.
- .NET integration tests MUST use Testcontainers to provision real infrastructure
  (PostgreSQL, Redis, message brokers, …); in-memory substitutes for those dependencies are
  not acceptable as integration tests.
- The Go operator MUST be tested with envtest (controller-runtime) for reconciliation logic.
- A bug fix MUST include a test that reproduces the bug.

Rationale: Requirements 5.1/5.2 demand unit and integration tests in CI; the gate that matters
is that tested code reaches `main`, while test-first stays a practice the developer chooses.

### IV. Blockchain on Sepolia, Tested Locally

- The target chain is Ethereum Sepolia testnet; users pay with MetaMask.
- Payments use a project-owned mock ERC20 token, mUSDT. Real-value tokens and mainnet are
  forbidden.
- Unit and integration tests involving contracts or chain interaction MUST run against a local
  node (Anvil or Hardhat). CI MUST NOT hit Sepolia.
- Sepolia MAY be used only for end-to-end tests and demos, to conserve testnet gas.
- Private keys and RPC credentials fall under Principle VI.

Rationale: deterministic, free and fast tests locally; a real testnet only where realism matters.

### V. Declarative Deployment: Platform in Git, Shops via API

Cluster state is split into two kinds, each with exactly one way in.

**Platform state** — operators, CRDs, the ShopHub application, the observability stack,
database operators (CloudNativePG, Redis operator) and all RBAC:

- MUST be deployed declaratively through Helm charts and recorded in `kube-state`
  (Principle I). The platform MUST be reproducible from `kube-state` plus the published charts.
- The shop-operator Helm chart MUST install the operator and its CRDs; the shophub chart MUST
  consume those CRDs and the prometheus-stack.

**Runtime state** — `Shop`, `Wallet`, `DiscordChannel` and database CRs (CloudNativePG
`Cluster`, Redis) created on behalf of ShopHub users:

- MUST be created, updated and deleted only by the ShopHub backend through the Kubernetes API.
- MUST NOT be stored in `kube-state` or any other git repository.
- The backend MUST run under a dedicated ServiceAccount whose RBAC grants only the verbs it
  needs on exactly these resource kinds, in the namespaces it manages. Wildcard resources or
  verbs and `cluster-admin` (or equivalent) bindings are forbidden.
- That ServiceAccount and its Role/RoleBinding (or ClusterRole, if justified in the plan) are
  platform state, defined in the shophub chart.
- Runtime CRs MUST carry labels identifying the owning user and shop. Restoring runtime state
  is the ShopHub backend's responsibility, not `kube-state`'s.

**For both**: raw `kubectl apply` / `kubectl create` / `kubectl edit` by a human is forbidden
for changing cluster state. `kubectl` MAY be used read-only for inspection and debugging.

Rationale: platform changes are reviewable and reproducible through git; user-driven shops are
dynamic data that cannot live in git, so they go through a narrowly-scoped API identity
instead.

### VI. Secrets Stay in Secret Stores

- Secrets (DB passwords, Discord tokens, wallet private keys, RPC keys, registry credentials,
  OAuth secrets) MUST live only in Kubernetes Secrets at runtime and GitHub Secrets in CI.
- Secrets MUST NOT be committed to any repository — including `values.yaml`, `kube-state`,
  test fixtures, specs or docs. Charts MUST reference secrets by name (`secretKeyRef`,
  `existingSecret`), never embed them.
- Local development MAY use untracked files (git-ignored) or well-known throwaway values
  for local-only tools (e.g. Anvil default accounts) that hold no value.
- A leaked secret MUST be rotated immediately; removing it from history alone is not enough.

Rationale: public repos plus blockchain keys make any leak immediately exploitable.

### VII. Per-Shop Observability

- Every Shop instance MUST emit metrics, logs and traces, provisioned automatically when the
  `Shop` resource is reconciled — no manual per-shop setup.
- Every Shop MUST get its own Grafana dashboard covering at least the Requirement 4.1 metrics:
  CPU, RAM, filesystem and network usage; total, successful (2xx/3xx) and failed (4xx/5xx)
  HTTP requests over 24 h; unique visitors; 404s per endpoint over 24 h; total traffic in GB.
- Each Shop MUST have alert rules that notify its own Discord channel.
- Cluster-level metrics (CPU, RAM, filesystem, network) and cluster alerts MUST also exist.
- Grafana access is restricted to maintainers; ShopHub end users MUST NOT see dashboards
  unless the optional per-owner access is implemented with correct authorization.

Rationale: Requirement 4.1; observability is a feature of every Shop, not an afterthought.

## Technology & Platform Constraints

- **Applications**: shophub and shop — Angular frontend + .NET backend.
- **Operator**: shop-operator — Go + kubebuilder.
- **Data**: PostgreSQL via CloudNativePG (`standard`), Redis via an operator (`light`).
- **Packaging/IaC**: Helm charts (OCI-published) in `helm-charts`; platform state in
  `kube-state`.
- **CI/CD**: GitHub Actions; images to a container registry with SemVer tags.
- **Observability**: kube-prometheus-stack (Prometheus, Alertmanager, Grafana) plus a log and
  trace backend; instrumentation via OpenTelemetry where practical.
- **Local cluster**: Docker Desktop Kubernetes.
- **Requirements source of truth**: `docs/ShopHub.md` (Serbian). Where this constitution and
  the requirements conflict, the requirements win unless the conflict is listed under
  "Recorded deviations" below; otherwise this constitution MUST be amended.
- Choosing technology outside this list requires a written justification in the feature's
  `plan.md` (Complexity Tracking).

## Development Workflow & Quality Gates

- **Recorded deviations** from `docs/ShopHub.md`:
  - **D-1 — Solo development vs. Requirement 5.1.** Requirement 5.1 requires every PR to be
    approved by at least one team member. The project has a single developer, so this
    requirement is **not met**; this is a known deviation, not compliance. Mitigations, both
    mandatory: (1) a self-review of the full PR diff before merge, and (2) required CI status
    checks that block merge on failure (Principle II). Every other rule in Requirement 5.1
    applies unchanged.
- **Spec-driven flow**: features go through specify → plan → tasks → implement in
  `shophub-specs`; each `plan.md` MUST include a Constitution Check against Principles I–VII.
- **CI on every PR** (shophub, shop, shop-operator): build, lint, unit tests, integration
  tests; on merge to `main`, build and publish a SemVer-tagged image.
- **Chart changes**: `helm lint` and `helm template` MUST pass in CI for `helm-charts`; chart
  versions MUST be bumped per SemVer on every change.
- **Definition of done**: unit and integration tests green in CI; platform changes deployed to
  the local cluster via Helm/kube-state; Shop-facing changes visible in that Shop's dashboard;
  no secrets in the diff.

## Governance

- This constitution supersedes other project practices and guidance. `CLAUDE.md` and other
  runtime guidance files MUST NOT contradict it.
- **Amendments**: proposed via PR to `shophub-specs` changing this file, stating the reason and
  any migration needed in affected repos. Amendments take effect on merge.
- **Versioning** (SemVer for this document):
  - MAJOR — a principle removed or redefined in a backward-incompatible way.
  - MINOR — a principle or section added, or guidance materially expanded.
  - PATCH — clarifications and wording fixes with no change in meaning.
- **Compliance**: every `plan.md` Constitution Check and every PR self-review MUST verify
  compliance. A feature-level deviation MUST be recorded with justification in the plan's
  Complexity Tracking table; a project-wide deviation from the requirements MUST be recorded
  under "Recorded deviations" in this constitution. Unrecorded deviations block merge.
  Principle I is NON-NEGOTIABLE and admits no deviation.
- **Review cadence**: revisit this constitution when the requirements document changes and
  before each project defense.

**Version**: 2.0.0 | **Ratified**: 2026-11-20 | **Last Amended**: 2026-11-20
