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

- [ ] T010 [P] [US1] Implement offline wallet logic in `shop-operator/internal/wallet/keygen.go` and `checksum.go` (generate key, derive checksummed address, EIP-55 validation for data-model.md `Wallet.spec.address`; research R10) with table tests in `shop-operator/internal/wallet/wallet_test.go` (known EIP-55 vectors, lowercase accepted, bad checksum rejected, key round-trip)
- [ ] T011 [P] [US1] Implement child builders in `shop-operator/internal/resources/` (`deployment.go`, `service.go`, `cnpg.go`, `redis.go` as unstructured) per research R11 and data-model.md "Owns" (names `<shop>`, `<shop>-db`, `<shop>-redis`; replicas from `Shop.spec.availability`; env `SHOP_NAME`, `WALLET_ADDRESS`, `DATABASE_URL` via secretKeyRef / `REDIS_URL`; owner labels) with unit tests in `shop-operator/internal/resources/resources_test.go`, including "image tag is never `latest`"
- [ ] T012 [US1] Implement the Wallet reconciler in `shop-operator/internal/controller/wallet_controller.go` (registered vs generated modes, never regenerate when `<name>-wallet-key` exists, `InvalidChecksum`, `KeyLost` when the Secret of an already-generated Wallet is gone, `usedBy` via the Shop index, finalizer `shophub.io/in-use-protection` with reason `InUse`) and envtest cases in `shop-operator/internal/controller/wallet_controller_test.go`, including "status, events and logs contain no private key" (SC-007) and "hand-deleting the `<name>-wallet-key` Secret of a generated Wallet whose `status.address` is set yields `Ready=False, reason=KeyLost`, keeps `status.address`, and creates no new key" (G4, FR-018 exception in spec Edge Cases); every not-ready reason is asserted with `Eventually(…, 60s)` (SC-005)
- [ ] T013 [US1] Write envtest cases for the Shop reconciler in `shop-operator/internal/controller/shop_controller_test.go` covering US1 acceptance scenarios 1–8 and edge cases: standard→PostgreSQL `Cluster`, light→`Redis`, availability change, wallet switch changes the pod template, deletion leaves owner references for GC, `WalletNotReady`/`DiscordChannelNotReady`, `DatabaseOperatorMissing` (manager started without the Redis kind), manual deletion of the Deployment is restored; every not-ready reason is asserted with `Eventually(…, 60s)` (SC-005)
- [ ] T014 [US1] Implement the Shop reconciler in `shop-operator/internal/controller/shop_controller.go` (resolve refs, create/update children with controller owner references, readiness per research R12, status `url`/`walletAddress`/`webhookSecretName`/replicas, watches on Wallet and DiscordChannel through the indexes) until T013 passes; declare `+kubebuilder:rbac` markers for every kind it touches, including `apps/statefulsets` `get;list;watch` for Redis readiness (research R12, R14 — envtest doesn't enforce RBAC)

**Checkpoint**: `make test` green; US1 behaviour proven in envtest

---

## Phase 4: User Story 3 - Each Shop can alert to its own Discord channel (Priority: P3)

**Goal**: DiscordChannel creates a channel and webhook on the project Discord server, stores the webhook in a Secret, and removes the channel on delete

**Independent Test**: envtest with an `httptest` fake Discord — channel created once (adopted by topic after a status loss), webhook Secret written, 429 retried, missing credentials reported, finalizer deletes the channel and waits while a Shop references it. Real Discord: quickstart Scenario 6

### Tests and implementation for User Story 3

- [ ] T015 [P] [US3] Implement the Discord client in `shop-operator/internal/discord/client.go` (interface + REST v10 implementation for the four calls in research R9, `Authorization: Bot`, `429 retry_after` → typed retryable error, 401/403 → forbidden, 404 on delete = success) with tests against `httptest` in `shop-operator/internal/discord/client_test.go`; the token must never appear in errors or logs
- [ ] T016 [US3] Implement the DiscordChannel reconciler in `shop-operator/internal/controller/discordchannel_controller.go` (token Secret from `--discord-token-secret`, read with the uncached `mgr.GetAPIReader()` because it is unlabelled and outside the filtered Secret cache from T009 (research R14), adopt by topic `shophub-uid:<uid>`, `<name>-discord-webhook` Secret with data-model.md `DiscordChannel.status.webhookSecretName`, reasons `DiscordCredentialsMissing`/`DiscordUnavailable`/`DiscordForbidden`/`InUse`, finalizer `shophub.io/discord-cleanup`) with envtest cases in `shop-operator/internal/controller/discordchannel_controller_test.go` using a fake client, covering US3 acceptance scenarios 1–4, "no webhook URL in status/events/logs", "an unlabelled token Secret is found", "a hand-deleted `<name>-discord-webhook` Secret is restored by creating a new webhook on the same channel, without creating a second channel" (G4, FR-018), and every not-ready reason asserted with `Eventually(…, 60s)` (SC-005)

**Checkpoint**: all three controllers green in CI; operator ready for its first release

---

## Phase 5: User Story 2 - Install the operator from kube-state (Priority: P2)

**Goal**: Operator image, `shop-operator`/`shophub-discord` charts 0.2.0 and kube-state releases (`cnpg`, `redis-operator`) so one bring-up provides everything

**Independent Test**: quickstart Scenario 2 — empty Docker Desktop cluster, `./bring-up.sh` → `5 installed`, CRDs Established, re-run `5 unchanged`; kube-state CI `e2e` green

### Implementation for User Story 2

- [ ] T017 [US2] Finish `shop-operator/Dockerfile` (distroless, non-root, pinned base digests) and `shop-operator/README.md` (CRDs, flags, wallet-deletion warning from spec Edge Cases, `KeyLost` recovery: the ShopHub backend creates a new Wallet (new key and address), switches each referencing Shop's `walletRef` to it (the shops roll to the new address), then deletes the old Wallet, which the in-use finalizer only allows once no Shop references it (FR-010); recreating a Wallet under the same name works only after that deletion completes; funds on the old address stay unreachable, Discord bot permissions and the one-time token Secret from research R15); apply branch protection to `nikola031ue/shop-operator-spec` `main` (contexts `lint`, `test`, `enforce_admins`, linear history, 0 approvals) **before** the first merge, then open, self-review and squash-merge the PR
- [ ] T018 [US2] Tag `v0.1.0` on `shop-operator` `main`, confirm the `release` job pushed `ghcr.io/nikola031ue/shop-operator:0.1.0`, set the package Public and verify an anonymous `docker pull`
- [ ] T019 [US2] Add `shop-operator/hack/sync-chart.sh` that renders `config/crd` and `config/rbac` into `helm-charts/charts/shop-operator/templates/crds.yaml` (with `helm.sh/resource-policy: keep`) and `rbac.yaml`; then write `helm-charts/charts/shop-operator/templates/{deployment,serviceaccount}.yaml`, `values.yaml` and bump `Chart.yaml` to version 0.2.0 / appVersion 0.1.0 exactly per contracts/charts-and-kube-state.md (research R8, R14)
- [ ] T020 [P] [US2] Replace the `helm-charts/charts/shophub-discord` placeholder with `templates/discordchannel.yaml` (one DiscordChannel, `channelName` from values, default `platform-alerts`) and bump to 0.2.0 (spec FR-025)
- [ ] T021 [US2] Change the `install` job in `helm-charts/.github/workflows/charts.yaml` to install `shop-operator` first and then the other charts; open the helm-charts PR, get all checks green (the chart must become Ready without CNPG/Redis or a Discord Secret), squash-merge, and confirm `publish` and `verify-public` push both 0.2.0 charts
- [ ] T022 [P] [US2] Add `kube-state/.github/upstream-charts.yaml` and the allowlist rule to `kube-state/.github/scripts/check-schema.sh` (every row of the contracts/charts-and-kube-state.md rule table), with bats cases in `kube-state/.github/tests/check-schema.bats` for: allowlisted url+version passes, allowlisted url with another tag fails, unknown url fails, missing/invalid allowlist fails, non-SemVer allowlist version fails (research R7)
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
  `cover.out` to `.gitignore`. Removed `.devcontainer/`. Kept kubebuilder's kind-based
  `test/e2e` scaffold (excluded from `make test`, not in CI). The old one-line README was
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
