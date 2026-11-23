---

description: "Task list for 002-shop-operator"
---

# Tasks: Shop Operator (Requirements 3.1, 3.2)

**Input**: Design documents from `/specs/002-shop-operator/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Required. Constitution Principle III mandates unit and envtest integration tests
before merge; the spec's acceptance scenarios are verified by them.

**Organization**: Tasks are grouped by user story. Story phases are ordered by **execution**
(US1 → US3 → US2), not priority number, because the single operator image released in US2 must
already contain the DiscordChannel controller from US3.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies on incomplete tasks)
- **[Story]**: US1 Shop, US2 install via chart/kube-state, US3 DiscordChannel

## Path Conventions

- `shop-operator/…` — new Go repo (`../shop-operator`, GitHub `nikola031ue/shop-operator-spec`)
- `helm-charts/…`, `kube-state/…` — existing repos from feature 001
- Field rules: cite [data-model.md](data-model.md) `Entity.field`; reason codes and admission
  messages: [contracts/crds.md](contracts/crds.md)

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Toolchain and operator repository skeleton

- [X] T001 Install Go on the maintainer machine (`winget install GoLang.Go`), start Docker Desktop, and record `go version` / `docker info` output in the implementation log at the end of this file (quickstart.md Prerequisites)
- [X] T002 Scaffold `shop-operator/` with kubebuilder v4: `kubebuilder init --domain io --repo github.com/nikola031ue/shop-operator`, then `kubebuilder create api --group shophub --version v1alpha1 --kind {Shop,Wallet,DiscordChannel} --resource --controller` (API group `shophub.io`, research R1–R2); keep the scaffolded Makefile, add `.gitattributes` (LF) and extend `.gitignore` (`bin/`, `cover.out`)
- [X] T003 [P] Add `shop-operator/hack/fetch-crds.sh` that writes the upstream CNPG `Cluster` CRD (chart 0.29.2, from `helm template`) and OT `Redis` CRD (chart 0.27.0, from `helm show crds`) into `shop-operator/test/crds/`, and commit the fetched files (research R4, R6)
- [X] T004 [P] Add `shop-operator/.github/workflows/ci.yaml`: jobs `lint` (`go vet`, golangci-lint) and `test` (`go test ./...` with `setup-envtest`) on PR; job `release` on tag `vX.Y.Z` building `linux/amd64` from `shop-operator/Dockerfile` and pushing `ghcr.io/nikola031ue/shop-operator:X.Y.Z` with `GITHUB_TOKEN` (`packages: write`); refuse tags that are not strict SemVer (research R16)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: API types, CRD validation, test harness and manager wiring used by every story

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [X] T005 [P] Define the Shop API in `shop-operator/api/v1alpha1/shop_types.go` with kubebuilder markers for data-model.md `Shop.metadata.name` (CEL length rule at the root), `Shop.spec.shopName`, `Shop.spec.availability`, `Shop.spec.database`, `Shop.spec.walletRef.name`, `Shop.spec.discordChannelRef.name` (immutability via CEL with the contracts/crds.md messages) and status fields `Shop.status.*`; printer columns per contracts/crds.md
- [X] T006 [P] Define the Wallet API in `shop-operator/api/v1alpha1/wallet_types.go` for data-model.md `Wallet.spec.address` (pattern + CEL immutable once set) and `Wallet.status.mode` (enum), plus `Wallet.status.address`, `keySecretName`, `usedBy`, conditions; printer columns per contracts/crds.md
- [X] T007 [P] Define the DiscordChannel API in `shop-operator/api/v1alpha1/discordchannel_types.go` for data-model.md `DiscordChannel.spec.channelName` (pattern) plus `DiscordChannel.status.channelID`, `webhookSecretName`, `usedBy`, conditions; printer columns per contracts/crds.md; then run `make manifests generate` so `shop-operator/config/crd/bases/` is current
- [X] T008 Add shared helpers in `shop-operator/internal/controller/common.go` (set `Ready` condition with reason constants from contracts/crds.md, `observedGeneration`, owner labels `shophub.io/owner`/`shophub.io/shop` per data-model.md) and an envtest suite in `shop-operator/internal/controller/suite_test.go` that loads `config/crd/bases` and `test/crds`; add `shop-operator/internal/controller/validation_test.go` asserting every admission rejection row of contracts/crds.md (SC-004)
- [X] T009 Wire `shop-operator/cmd/main.go`: flags `--shop-image`, `--shop-image-tag`, `--discord-guild-id`, `--discord-token-secret` (data-model.md "Platform configuration"); discovery check that only registers `Owns()` for served database kinds (research R5); Secret cache restricted to label `app.kubernetes.io/managed-by=shop-operator` (research R14); field indexes `spec.walletRef.name` and `spec.discordChannelRef.name` (research R13)

**Checkpoint**: `make test` passes with the validation tests; CRDs generate cleanly

---

## Phase 3: User Story 1 - Declare a Shop and get a running shop (Priority: P1) 🎯 MVP

**Goal**: Shop + Wallet reconcile into a running placeholder app with its database and wallet address; updates and deletion follow the Shop

**Independent Test**: envtest — create a Wallet, a DiscordChannel whose status the test sets to Ready (no DiscordChannel controller needed), and a Shop; assert children, env, replicas 2→3, wallet switch and readiness reasons. On a real cluster after US2: quickstart Scenarios 3–5

### Tests and implementation for User Story 1

- [X] T010 [P] [US1] Implement offline wallet logic in `shop-operator/internal/wallet/keygen.go` and `checksum.go` (generate key, derive checksummed address, EIP-55 validation for data-model.md `Wallet.spec.address`; research R10) with table tests in `shop-operator/internal/wallet/wallet_test.go` (known EIP-55 vectors, lowercase accepted, bad checksum rejected, key round-trip)
- [X] T011 [P] [US1] Implement child builders in `shop-operator/internal/resources/` (`deployment.go`, `service.go`, `cnpg.go`, `redis.go` as unstructured) per research R11 and data-model.md "Owns" (names `<shop>`, `<shop>-db`, `<shop>-redis`; replicas from `Shop.spec.availability`; env `SHOP_NAME`, `WALLET_ADDRESS`, `DATABASE_URL` via secretKeyRef / `REDIS_URL`; owner labels) with unit tests in `shop-operator/internal/resources/resources_test.go`, including "image tag is never `latest`"
- [X] T012 [US1] Implement the Wallet reconciler in `shop-operator/internal/controller/wallet_controller.go` (registered vs generated modes, never regenerate when `<name>-wallet-key` exists, `InvalidChecksum`, `KeyLost` when the Secret of an already-generated Wallet is gone, `usedBy` via the Shop index, finalizer `shophub.io/in-use-protection` with reason `InUse`) and envtest cases in `shop-operator/internal/controller/wallet_controller_test.go`, including "status, events and logs contain no private key" (SC-007) and "hand-deleting the `<name>-wallet-key` Secret of a generated Wallet whose `status.address` is set yields `Ready=False, reason=KeyLost`, keeps `status.address`, and creates no new key" (G4, FR-018 exception in spec Edge Cases); every not-ready reason is asserted with `Eventually(…, 60s)` (SC-005)
- [X] T013 [US1] Write envtest cases for the Shop reconciler in `shop-operator/internal/controller/shop_controller_test.go` covering US1 acceptance scenarios 1–8 and edge cases: standard→PostgreSQL `Cluster`, light→`Redis`, availability change, wallet switch changes the pod template, deletion leaves owner references for GC, `WalletNotReady`/`DiscordChannelNotReady`, `DatabaseOperatorMissing` (manager started without the Redis kind), manual deletion of the Deployment is restored; every not-ready reason is asserted with `Eventually(…, 60s)` (SC-005)
- [X] T014 [US1] Implement the Shop reconciler in `shop-operator/internal/controller/shop_controller.go` (resolve refs, create/update children with controller owner references, readiness per research R12, status `url`/`walletAddress`/`webhookSecretName`/replicas, watches on Wallet and DiscordChannel through the indexes) until T013 passes; declare `+kubebuilder:rbac` markers for every kind it touches, including `apps/statefulsets` `get;list;watch` for Redis readiness (research R12, R14 — envtest doesn't enforce RBAC)

**Checkpoint**: `make test` green; US1 behaviour proven in envtest

---

## Phase 4: User Story 3 - Each Shop can alert to its own Discord channel (Priority: P3)

**Goal**: DiscordChannel creates a channel and webhook on the project Discord server, stores the webhook in a Secret, and removes the channel on delete

**Independent Test**: envtest with an `httptest` fake Discord — channel created once (adopted by topic after a status loss), webhook Secret written, 429 retried, missing credentials reported, finalizer deletes the channel and waits while a Shop references it. Real Discord: quickstart Scenario 6

### Tests and implementation for User Story 3

- [X] T015 [P] [US3] Implement the Discord client in `shop-operator/internal/discord/client.go` (interface + REST v10 implementation for the four calls in research R9, `Authorization: Bot`, `429 retry_after` → typed retryable error, 401/403 → forbidden, 404 on delete = success) with tests against `httptest` in `shop-operator/internal/discord/client_test.go`; the token must never appear in errors or logs
- [X] T016 [US3] Implement the DiscordChannel reconciler in `shop-operator/internal/controller/discordchannel_controller.go` (token Secret from `--discord-token-secret`, read with the uncached `mgr.GetAPIReader()` because it is unlabelled and outside the filtered Secret cache from T009 (research R14), adopt by topic `shophub-uid:<uid>`, `<name>-discord-webhook` Secret with data-model.md `DiscordChannel.status.webhookSecretName`, reasons `DiscordCredentialsMissing`/`DiscordUnavailable`/`DiscordForbidden`/`InUse`, finalizer `shophub.io/discord-cleanup`) with envtest cases in `shop-operator/internal/controller/discordchannel_controller_test.go` using a fake client, covering US3 acceptance scenarios 1–4, "no webhook URL in status/events/logs", "an unlabelled token Secret is found", "a hand-deleted `<name>-discord-webhook` Secret is restored by creating a new webhook on the same channel, without creating a second channel" (G4, FR-018), and every not-ready reason asserted with `Eventually(…, 60s)` (SC-005)

**Checkpoint**: all three controllers green in CI; operator ready for its first release

---

## Phase 5: User Story 2 - Install the operator from kube-state (Priority: P2)

**Goal**: Operator image, `shop-operator`/`shophub-discord` charts 0.2.0 and kube-state releases (`cnpg`, `redis-operator`) so one bring-up provides everything

**Independent Test**: quickstart Scenario 2 — empty Docker Desktop cluster, `./bring-up.sh` → `5 installed`, CRDs Established, re-run `5 unchanged`; kube-state CI `e2e` green

### Implementation for User Story 2

- [X] T017 [US2] Finish `shop-operator/Dockerfile` (distroless, non-root, pinned base digests) and `shop-operator/README.md` (CRDs, flags, wallet-deletion warning from spec Edge Cases, `KeyLost` recovery: the ShopHub backend creates a new Wallet (new key and address), switches each referencing Shop's `walletRef` to it (the shops roll to the new address), then deletes the old Wallet, which the in-use finalizer only allows once no Shop references it (FR-010); recreating a Wallet under the same name works only after that deletion completes; funds on the old address stay unreachable, Discord bot permissions and the one-time token Secret from research R15); apply branch protection to `nikola031ue/shop-operator-spec` `main` (contexts `lint`, `test`, `enforce_admins`, linear history, 0 approvals) **before** the first merge, then open, self-review and squash-merge the PR
- [X] T018 [US2] Tag `v0.1.0` on `shop-operator` `main`, confirm the `release` job pushed `ghcr.io/nikola031ue/shop-operator:0.1.0`, set the package Public and verify an anonymous `docker pull`
- [X] T019 [US2] Add `shop-operator/hack/sync-chart.sh` that renders `config/crd` and `config/rbac` into `helm-charts/charts/shop-operator/templates/crds.yaml` (with `helm.sh/resource-policy: keep`) and `rbac.yaml`; then write `helm-charts/charts/shop-operator/templates/{deployment,serviceaccount}.yaml`, `values.yaml` and bump `Chart.yaml` to version 0.2.0 / appVersion 0.1.0 exactly per contracts/charts-and-kube-state.md (research R8, R14)
- [X] T020 [P] [US2] Replace the `helm-charts/charts/shophub-discord` placeholder with `templates/discordchannel.yaml` (one DiscordChannel, `channelName` from values, default `platform-alerts`) and bump to 0.2.0 (spec FR-025)
- [X] T021 [US2] Change the `install` job in `helm-charts/.github/workflows/charts.yaml` to install `shop-operator` first and then the other charts; open the helm-charts PR, get all checks green (the chart must become Ready without CNPG/Redis or a Discord Secret), squash-merge, and confirm `publish` and `verify-public` push both 0.2.0 charts
- [X] T022 [P] [US2] Add `kube-state/.github/upstream-charts.yaml` and the allowlist rule to `kube-state/.github/scripts/check-schema.sh` (every row of the contracts/charts-and-kube-state.md rule table), with bats cases in `kube-state/.github/tests/check-schema.bats` for: allowlisted url+version passes, allowlisted url with another tag fails, unknown url fails, missing/invalid allowlist fails, non-SemVer allowlist version fails (research R7)
- [ ] T023 [US2] Add `kube-state/clusters/local/cnpg/` and `kube-state/clusters/local/redis-operator/` (`helm.yaml`, `values.yaml`), update `kube-state/clusters/local/cluster.yaml` `releaseOrder`, bump `shop-operator` (0.2.0, `dependsOn` cnpg + redis-operator) and `shophub-discord` (0.2.0) per contracts/charts-and-kube-state.md; document the Discord token bootstrap and the new releases in `kube-state/README.md`; before opening the PR, render both upstream releases locally (`helm template` with their `values.yaml`) and run `kube-state/.github/scripts/check-images.sh` and `check-schema.sh` on the result so upstream image tags pass the repo rules; open the kube-state PR, get `charts-exist`, `render`, `e2e` green, squash-merge

**Checkpoint**: bring-up from kube-state installs the whole platform including the operator

---

## Phase 6: Polish & Cross-Cutting Concerns

- [ ] T024 Prerequisite: a Discord server, bot token and guild ID (quickstart Prerequisites). First run quickstart Scenario 2a step 1 (upgrade from the feature 001 releases), then reset Kubernetes and run Scenarios 1, 2, 6, 3, 2a step 2, 4, 5, 7 in that order on Docker Desktop; record timings for SC-001, SC-002, SC-003, SC-005, SC-006 and results in a "Validation log" section of `specs/002-shop-operator/quickstart.md`; delete namespace `qs` afterwards
- [ ] T025 [P] Self-review all three merged diffs against the plan's Constitution Check and Complexity Tracking; grep all repos for secret-looking values (SC-007); record the outcome and any deviations in the implementation log at the end of `specs/002-shop-operator/tasks.md`

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (1)** → **Foundational (2)** → US1 (3) → US3 (4) → US2 (5) → **Polish (6)**
- T003 must finish before T008 (envtest loads `test/crds`)

### User Story Dependencies

- **US1 (P1)**: needs Foundational only. Its envtest tests set the DiscordChannel status
  directly, so no DiscordChannel controller is required.
- **US3 (P3)**: needs Foundational only; it doesn't depend on US1. It runs before US2 so that
  the first image contains it.
- **US2 (P2)**: needs US1 and US3 merged in `shop-operator` (T017 releases the code). Inside US2:
  - T018 → T019/T021: the image must be public before the chart `install` job.
  - T021 → T023: the charts must be published before kube-state `charts-exist`.
  - T022 can be done any time after Foundational.

### Within Each Story

- T013 (tests) before T014 (Shop reconciler); T010/T011 before T012/T014
- Branch protection (in T017) before the first `shop-operator` merge — Principle II

### Parallel Opportunities

- T003 ∥ T004 (Setup)
- T005 ∥ T006 ∥ T007 (separate type files)
- T010 ∥ T011, and T015 can start alongside them (separate packages)
- T020 ∥ T019; T022 ∥ everything in US2 before T023

---

## Parallel Example: User Story 1

```bash
Task: "T010 Offline wallet logic + tests in shop-operator/internal/wallet/"
Task: "T011 Child builders + tests in shop-operator/internal/resources/"
Task: "T015 Discord client + httptest tests in shop-operator/internal/discord/"   # US3, independent package
```

---

## Implementation Strategy

### MVP First (User Story 1)

1. Phase 1 + Phase 2 → CRDs with validation, envtest harness
2. Phase 3 → Shop + Wallet proven in envtest (**stop and validate**: `make test`)
3. MVP isn't released alone: the first image (T018) also includes US3, to avoid a second
   chart/kube-state round-trip.

### Incremental Delivery

1. `shop-operator` PR (US1 + US3) → image 0.1.0
2. `helm-charts` PR (charts 0.2.0) → published
3. `kube-state` PR (allowlist, database operators, bumps) → platform bring-up
4. Quickstart validation (T024)

---

## Notes

- 25 tasks, sized for a solo developer; commit per task with Conventional Commits.
- Never commit the Discord token, generated keys or webhook URLs (Principle VI).
- Manual CR applies are allowed only in namespace `qs` during T024 (plan Complexity Tracking).

## Implementation log

### 2026-11-23 — Phase 1–2 (T001–T009)

- **T001**: Go was already installed (`winget` found `GoLang.Go`): `go version go1.27.4
  windows/amd64`. Docker Desktop running: `docker info` → server 29.9.3. kubebuilder ships no
  Windows binary and envtest is most reliable on Linux, so scaffolding, tests and lint run in a
  local dev image `shop-operator-dev:local` (`golang:1.27-bookworm` + kubebuilder v4.16.0; not
  committed). CI runs the same commands on `ubuntu-latest`.
- **T002**: kubebuilder **v4.16.0** (released 2026-09-10) scaffold on branch `002-shop-operator`
  in `shop-operator`: controller-runtime v0.25.0, k8s.io v0.37.0, controller-tools v0.22.0,
  `go 1.26.0` directive, API group `shophub.io/v1alpha1`. Added `.gitattributes` (LF) and
  `cover.out` to `.gitignore`. Removed `.devcontainer/` and, on request, the kind-based
  `test/e2e` + `test/utils` scaffold and its Makefile targets (`setup-test-e2e`, `test-e2e`,
  `cleanup-test-e2e`, `KIND`); `make test` now runs `go test ./...`; AGENTS.md updated. The old one-line README was
  replaced by kubebuilder's; T017 rewrites it.
- **T003**: `hack/fetch-crds.sh` wrote `test/crds/postgresql.cnpg.io_clusters.yaml` (v1, chart
  0.29.2) and `test/crds/redis.redis.opstreelabs.in_redis.yaml` (v1beta2, chart 0.27.0).
- **T004**: kubebuilder's `lint.yml`, `test.yml`, `test-e2e.yml` replaced by
  `.github/workflows/ci.yaml` (jobs `lint`, `test`, `release`). Actions are pinned by SHA. The
  `test` job also fails if `make manifests generate` changes `api/` or `config/`. `release`
  refuses non-SemVer tags and existing image versions.
- **T005–T007**: types per data-model.md, with CEL rules for immutability, the 40-char Shop
  name, and Wallet address set-once; printer columns per contracts/crds.md.
- **T008**: `internal/controller/common.go` (reason constants incl. `KeyLost`, `setReady`,
  `childLabels`). The suite loads `config/crd/bases` + `test/crds`. `validation_test.go` covers
  every admission row of contracts/crds.md. The scaffolded per-controller tests were removed
  (their empty specs now fail validation); T012/T013/T016 recreate them.
- **T009**: `cmd/main.go`:
  - flags `--shop-image`, `--shop-image-tag`, `--discord-guild-id`, `--discord-token-secret`;
  - token Secret namespace from env `POD_NAMESPACE` (**T019: the chart must set it via the
    downward API**);
  - `Config.Validate` refuses an empty or `latest` image tag;
  - Secret cache filtered on `app.kubernetes.io/managed-by=shop-operator`;
  - `DetectDatabaseKinds` via discovery;
  - Shop field indexes;
  - leader election ID `shop-operator.shophub.io`; zap production mode.
  - Reconciler structs carry `Config`/`DBKinds`/`APIReader`.
  - Tests: config validation, discovery with and without the upstream CRDs (second envtest
    API server), and indexes through a real manager.
- **Deviation**: the scaffold pinned golangci-lint **v2.13.1**, which cannot read Go 1.27
  export data (`export data version 5 is greater than maximum supported version 4`). Bumped to
  **v2.15.0** in the `Makefile` and `.custom-gcl.yml`. The `logcheck` plugin was `version:
  latest`; it is now pinned to **v0.11.0**.
- **Checkpoint**: `make test` → 27/27 Ginkgo specs pass, `internal/controller` coverage 71.2%;
  `make lint` → 0 issues; `go build ./cmd` OK. Nothing committed yet (shop-operator branch
  `002-shop-operator` is local).

### 2026-11-23 — Phase 3 / US1 (T010–T014)

- **T010** `internal/wallet`: `GenerateKey`, `AddressFromPrivateKeyHex`, `NormalizeAddress`
  (EIP-55; all-lower/all-upper accepted, mixed case must match). Dependency
  `github.com/ethereum/go-ethereum` **v1.17.10** (2026-10-23). `go get` also bumped transitive
  `google.golang.org/grpc` 1.82.1 → 1.83.1 and `google.golang.org/protobuf` to the 1.36.12
  release. 5 tests (EIP-55 reference vectors, bad checksum, invalid formats, key round trip, no
  key material in errors), 94.4% coverage.
- **T011** `internal/resources`: all builders in one `resources.go` instead of the four files
  the task names (small package). Redis server image pinned to
  `quay.io/opstree/redis:v7.2.16`, because the upstream example uses `latest`. CNPG's PostgreSQL
  image is left to the pinned operator's default. Owner labels stay out of the immutable
  Deployment selector. 8 tests, 100% coverage.
- **T012** Wallet reconciler:
  - registered/generated modes, `InvalidChecksum`, `KeyLost`, `usedBy`, finalizer
    `shophub.io/in-use-protection` (`InUse`).
  - Race handling: before reporting `KeyLost`, a cache miss on the key Secret is confirmed with
    the **uncached API reader** (the cache can lag right after creation). `AlreadyExists` on
    create requeues instead of overwriting.
  - The Wallet watches Shops; update events map both the old and new `walletRef`.
  - `CacheOptions()` is now shared by `cmd/main.go` and the test manager.
  - 8 envtest specs, including no private key in status or logs (log capture) and G4.
- **T013** Shop envtest specs written first and confirmed failing (7/7 red) before T014.
  envtest has no kube-controller-manager, CNPG or Redis operator, so tests set Deployment, CNPG
  `Cluster` and Redis StatefulSet status by hand. Garbage collection doesn't run either, so
  deletion (scenario 5) is verified via controller owner references with `blockOwnerDeletion`.
  `DatabaseOperatorMissing` is tested against a second, bare API server with the reconciler
  called directly.
- **T014** Shop reconciler:
  - no app until Wallet and DiscordChannel are Ready;
  - children are server-side applied (`Client.Apply` + `ApplyConfigurationFromUnstructured`;
    the Patch-based `client.Apply` is deprecated in controller-runtime v0.25) with field owner
    `shop-operator` and controller owner references;
  - readiness per research R12, 30 s requeue while not Ready.
  - RBAC markers include `apps/statefulsets get;list;watch`.
- **RBAC tightened**: the scaffold granted `create`/`delete` on Shops, Wallets and
  DiscordChannels. The operator never creates user resources (Principle V), so those verbs and
  `shops/finalizers` are gone. `config/rbac/role.yaml` regenerated.
- **Test fixes**: once the Shop reconciler writes status, plain `Update` calls in tests can hit
  conflicts. Validation tests now use JSON merge patches; the spec updates in the Shop/Wallet
  tests retry.
- **Checkpoint**: `make test` → 42/42 Ginkgo specs (two consecutive green runs), controller
  coverage ≈ 83.5%; `make lint` → 0 issues after fixing 12 findings (incl. the deprecated
  `Result.Requeue` → `RequeueAfter`).

### 2026-11-23 — Phase 4 / US3 (T015–T016)

- **T015** `internal/discord`: `Client` interface plus `HTTPClient` over REST v10:
  - list/create channel, create webhook, delete channel;
  - `Authorization: Bot`, `DiscordBot (…)` User-Agent;
  - 429 → `RetryableError` with `retry_after` (body, else `Retry-After` header); 5xx and
    network errors are retryable too; 401/403 → `ForbiddenError`; 404 on delete = success;
  - errors carry only the operation, status and Discord's code/message, never the token or
    webhook URL.
  - 9 `httptest` tests, 92.1% coverage.
- **T016** DiscordChannel reconciler:
  - finalizer `shophub.io/discord-cleanup`;
  - adopts by topic `shophub-uid:<uid>` when `status.channelID` is empty, otherwise creates;
  - webhook Secret `<name>-discord-webhook` (key `url`, owned, labelled); like the wallet key,
    a cache miss is confirmed with the uncached reader before creating a new webhook;
  - reasons `DiscordCredentialsMissing` (names the token Secret), `DiscordUnavailable`
    (requeue after `retry_after`, at least 1 s), `DiscordForbidden` (5 min), `InUse`.
  - The unlabelled token Secret is read with `mgr.GetAPIReader()`. It isn't watched, so missing
    credentials are re-checked every `CredentialsRetry` (default 1 min).
  - The real HTTP client is the default in `SetupWithManager`, so `cmd/main.go` needs no change.
- **Tests**:
  - Each DiscordChannel spec runs its own manager, limited to its namespace, with an in-memory
    fake Discord. A suite-wide DiscordChannel reconciler would overwrite the statuses the US1
    tests set by hand. Repeated controller names need `SkipNameValidation` in those test
    managers.
  - 10 specs cover US3 scenarios 1–4, adoption after status loss, G4 (webhook Secret restored
    without a second channel), in-use protection, both credential failures, and no
    token/webhook URL in status or logs.
- **Checkpoint**: `make test` → 52/52 Ginkgo specs (two consecutive green runs), controller
  coverage 83.5%; `make lint` → 0 issues.

### 2026-11-23 — Phase 5 part 1 / US2 (T017–T018)

- **T017**:
  - `Dockerfile`: builder `golang:1.27.4-bookworm` and runtime
    `gcr.io/distroless/static:nonroot`, both pinned by digest; `-trimpath -ldflags="-s -w"`,
    user 65532. Image 80 MB (scaffold build: 120 MB).
  - `README.md` rewritten: resources, wallet key handling with the wallet-deletion warning and
    the `KeyLost` recovery (new Wallet → move Shops → delete old one; same-name reuse only after
    deletion completes), Discord bot setup and the one-time token Secret, flags, development,
    releases.
  - Branch protection on `nikola031ue/shop-operator-spec` `main` applied **before** the first
    merge, after approval: required checks `lint`, `test` (strict), `enforce_admins`, linear
    history, no force-push or deletion, 0 approvals. Repo set to squash-only with auto-delete
    of merged branches.
  - PR #1: CI green (`lint` 3m56s, `test` 3m52s), mergeable `CLEAN`, diff self-reviewed and
    secret-scanned, squash-merged as `687f75c`.
- **T018**:
  - Tag `v0.1.0` on `687f75c`; the `release` run passed `lint`, `test` and `release`. Published
    `ghcr.io/nikola031ue/shop-operator:0.1.0`, digest
    `sha256:a229778fd983d834fe7cb9a844228e90c4eab4b3326a5f36324c748de70001d9`, linux/amd64,
    OCI labels `version=0.1.0` and `source=…/shop-operator-spec`.
  - **The package was already Public**, so no visibility change was needed or made. Probably it
    inherited the public repository's visibility through the source label on first publish.
    The `gh` token lacks `read:packages`, so the setting couldn't be read via the API.
    Evidence: GHCR issues an anonymous pull token (HTTP 200; it is 403 for private or missing
    packages, see 001 research R9).
  - A fresh pull with an empty `DOCKER_CONFIG` (no credentials) succeeded. The pulled image runs
    as 65532 and refuses `--shop-image-tag=latest`.
- Remaining in Phase 5: T019–T023 (helm-charts and kube-state).

### 2026-11-23 — Phase 5 part 2 / US2 (T019–T023)

- **Deviation (value name)**: the chart value `discord.tokenSecret` from the contract would trip
  the helm-charts secrets scan (key matches "token"/"secret" and isn't a by-name reference).
  Renamed to **`discord.tokenSecretName`** in the chart, the contract, data-model.md,
  research.md and the operator README.
- **T019**:
  - `shop-operator/hack/sync-chart.sh` renders `config/crd/bases` into
    `templates/crds.yaml` (`helm.sh/resource-policy: keep`; refuses CRDs containing `{{`) and
    the generated ClusterRole plus leader-election Role and their bindings into
    `templates/rbac.yaml`. Merged as shop-operator-spec#2 (`e1b3da8`) after approval; no new
    image.
  - Chart `shop-operator` 0.2.0 / appVersion 0.1.0: Deployment (leader election, non-root,
    read-only root fs, seccomp RuntimeDefault, drop ALL, `/healthz` `/readyz` on :8081,
    `POD_NAMESPACE` from the downward API), ServiceAccount, values `image`, `shopImage`,
    `discord.guildID`, `discord.tokenSecretName`, `resources`; NOTES warn when Discord is
    unconfigured.
- **T020**: `shophub-discord` 0.2.0 — `templates/discordchannel.yaml` (`platform-alerts`).
- **T021**:
  - `install` job installs `shop-operator` first, waits for the three CRDs, then the rest, and
    checks the operator rolls out.
  - Local toolbox: layout, lint ×4, kubeconform, image check, secrets scan, bats 52/52.
  - helm-charts-spec#5: all 5 checks green; on kind the operator "successfully rolled out"
    without CNPG/Redis/Discord.
  - Merged after approval as `5db7710`; `publish` + `verify-public` green; both 0.2.0 charts
    anonymously pullable.
- **T022**: `.github/upstream-charts.yaml` + allowlist rule in `check-schema.sh` (all contract
  rows incl. duplicates and non-`oci://`); fixture writes an allowlist; 7 new bats cases →
  92/92.
- **T023**:
  - Releases `cnpg` and `redis-operator` added; `releaseOrder` cnpg → redis-operator →
    shop-operator → shophub → shophub-discord; `shop-operator` dependsOn both.
  - Bumps to 0.2.0; e2e expects 5 releases; README covers the upstream charts and the Discord
    token bootstrap.
  - R1: all releases rendered locally and checked (CNPG `1.30.2`, OT `v0.27.0`); layout,
    schema and secrets OK.
  - kube-state-spec#3 opened; **not merged** (waiting for approval).
- **Bug found by kube-state e2e (feature 001 `bring-up.sh`)**:
  - The CRD wait passed helm's `--kube-context` to `kubectl wait`. kubectl only knows
    `--context`, so every wait failed instantly ("unknown flag") and was misreported as a 120 s
    timeout (exit 5).
  - It was latent since 001: all 001 charts were placeholders without CRDs, and the kubectl test
    stub accepted any flag. CNPG's real CRDs exposed it on the first e2e run of
    kube-state-spec#3.
  - Fix `c631db5`: separate `KC` (helm) and `KCTL` (kubectl) flags; kubectl's own error is now
    printed. The kubectl stub rejects `--kube-context` like the real binary (3 tests turned red
    first, as reproduction); 2 regression tests added → bats 94/94.
- kube-state-spec#3 CI after the fix: all 7 checks green. e2e: `5 installed` with the CNPG,
  Redis and shophub.io CRDs `Established`, re-run `5 unchanged`, wrong context refused.
  **Not merged** — waiting for approval (T023 stays open until then).
