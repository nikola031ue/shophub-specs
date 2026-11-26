# Tasks: Crypto Payments (Requirement 2.4)

**Input**: `specs/005-crypto-payments/` — plan.md, spec.md, research.md (R1–R14), data-model.md,
contracts/ (shop-api, shop-internal-api, operator, musdt), quickstart.md (M1–M4, Scenarios 1–10)

**Tests**: required by constitution Principle III, so each implementation task includes its unit
and integration tests (Testcontainers in shop; envtest + Anvil in shop-operator; k3s in shophub).
CI never contacts Sepolia (Principle IV).

**Organization**: phases follow the maintainer's delivery order (one component release per
phase); every task carries the user story it serves. **Commit at the end of each phase.** Every
merge, tag, release, chart publish and kube-state PR **asks the maintainer first**.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: can run in parallel (different files or repos, no dependency on an open task)
- **[Story]**: US1 pay · US2 expiry · US3 payment paths and D-1/D-4/D-5/D-6 · US4 admin view
- **[MANUAL]**: done by the maintainer by hand; the agent only checks the result (never sees or
  prints a key or URL)

## Path Conventions

Repos are siblings of `shophub-specs`: `../shop`, `../shop-operator`, `../shophub`,
`../helm-charts`, `../kube-state`. Paths below are relative to the named repo.

---

## Phase 0: Manual steps + token (Setup)

**Purpose**: everything the code needs from outside, and the token every later test uses.

- [ ] T001 [MANUAL] **M1 provider account**: maintainer creates a free Sepolia endpoint at Infura
  or Alchemy (quickstart M1). Done when the maintainer confirms an HTTPS endpoint URL with key
  exists; nothing is recorded anywhere.
- [ ] T002 [MANUAL] **M2 RPC Secret**: maintainer runs
  `kubectl create secret generic shop-operator-rpc -n shop-operator-system --from-literal=url="$SEPOLIA_RPC_URL"`
  on docker-desktop (quickstart M2; data-model.md "Platform settings and Secrets"
  `payments.rpcSecretName`). Agent checks only that key `url` is non-empty (prints its length).
- [ ] T003 Token source and tests in shop-operator (contracts/musdt.md, research R8):
  `contracts/MockUSDT.sol` (OpenZeppelin ERC20 pinned, 6 decimals, `mint` capped at 1,000 mUSDT,
  `MintTooLarge()`); compiled `contracts/build/MockUSDT.{abi,bin}` with a pinned `solc`;
  `contracts/README.md` (compiler version, optimizer settings, Remix steps, deployed address
  placeholder); CI step in `.github/workflows/ci.yaml` that recompiles and fails on a diff and a
  step that fails if any workflow or test setting names a Sepolia/provider host (only
  shop-operator reads the chain; shop and shophub have no chain access, so SC-008's "0 CI runs
  contact Sepolia" is checked there); Go test
  `internal/chain/token_test.go` that starts Anvil (testcontainers-go, new dependency), deploys
  the committed bytecode and checks mint cap, `transfer`, and the `Transfer` event topic. Branch
  `005-token`, PR, **ask before merge**.
- [ ] T004 [MANUAL] **M3 Remix deploy** (only after T003 is merged, so the deployed source is the
  reviewed, tested one): maintainer deploys `contracts/MockUSDT.sol` from `main` on Sepolia
  from their own MetaMask with the settings in `contracts/README.md`, and verifies the source on
  the explorer (quickstart M3). Agent records the address in `contracts/README.md` (PR, ask before
  merge) and in this file; it goes into kube-state in T024.
- [ ] T005 [MANUAL] **M4 tester wallets**: maintainer prepares MetaMask accounts A and B with
  Sepolia ETH from a public faucet (quickstart M4). Needed by Phase 4 only.

**Checkpoint**: token on Anvil in CI; address known; Secret present. Commit (spec repo).

---

## Phase 1: Shop app 0.3.0 (US1, US2, US4) — repo `shop`

**Goal**: orders carry a paying account and deadline; the shop receives transfers on its internal
port, matches or records them, expires orders behind the chain, and shows payment in the
storefront and admin.
**Independent test**: with the operator replaced by test calls to `/internal/chain`, an exact
transfer marks an order Paid once, every wrong one is recorded with its reason, expiry returns
stock once (quickstart Scenario 1, shop part).

- [ ] T006 [US1] Settings and schema: `src/Shop.Api/Settings.cs` reads `TOKEN_ADDRESS`, `CHAIN_ID`,
  `EXPLORER_TX_URL`, `ORDER_PAYMENT_MINUTES` (default 30), `INTERNAL_TOKEN` (all optional; missing
  token address → payments unavailable, missing internal token → port 8081 not opened);
  `src/Shop.Api/Storage/Postgres/Scripts/004_payments.sql` and the Redis layout per data-model.md
  `Order.paying_account`, `Order.total_base_units`, `Order.deadline`, `Order.status`,
  `Order.paid_tx`/`paid_log_index`/`paid_block`/`paid_at`; `Payment.tx_hash` + `Payment.log_index`
  (primary key), `Payment.block`, `Payment.block_time`, `Payment.from_account`,
  `Payment.to_address`, `Payment.value_base_units`, `Payment.order_reference`,
  `Payment.unmatched_reason`, `Payment.received_at`; `Chain progress.address`,
  `Chain progress.seen_through_block`, `Chain progress.seen_through_time`; existing orders migrated
  as data-model.md says.
  Tests: migration on an 0.2.0 database, settings validation.
- [ ] T007 [US1] Checkout and order lookup (contracts/shop-api.md "Public"): `POST /api/orders`
  takes `payingAccount` (EIP-55, data-model.md `Order.paying_account`), stores
  `total_base_units = cents × 10^4` and `deadline`; the one-waiting-order rule (data-model.md
  "Uniqueness (FR-005a)": PostgreSQL partial unique index, Redis NX key in
  `src/Shop.Api/Storage/Redis/Scripts/place_order.lua`) answering 409 `waitingOrder` with the
  reference; 409 `paymentsUnavailable`; `GET /api/orders/{reference}`; `GET /api/shop` `payments`
  block. Files: `src/Shop.Api/Endpoints/Public.cs`, `src/Shop.Api/Storage/IShopStore.cs`, both
  stores. Tests on both stores, including two concurrent checkouts with one account (one wins).
- [ ] T008 [US1] Internal endpoint (contracts/shop-internal-api.md): second Kestrel endpoint on
  8081 in `src/Shop.Api/Program.cs`, `src/Shop.Api/Endpoints/Internal.cs` with constant-time bearer
  check against `INTERNAL_TOKEN`; `/internal/*` answers 404 on 8080. Tests: 401 without/with wrong
  token, 404 on the public port, 400 for a foreign address.
- [ ] T009 [US1] Matching (research R6; contracts/shop-internal-api.md "POST /internal/chain"):
  `src/Shop.Api/Payments/Matching.cs`; PostgreSQL in one transaction per transfer, Redis
  `src/Shop.Api/Storage/Redis/Scripts/record_transfer.lua`; idempotent on (`tx`, `logIndex`);
  sets data-model.md `Payment.order_reference` on a match, otherwise `Payment.unmatched_reason`
  (`amount`, `account`, `late`, `duplicate`, `unknown`); clears the waiting key on Paid; `chain_progress` only moves forward
  (data-model.md `Chain progress.seen_through_block`/`seen_through_time`); 204 only when all is
  stored. Tests on both stores: each reason, redelivery of the same range, two shop copies
  receiving the same transfer at once (Paid once).
- [ ] T010 [US2] Expiry and waiting count (research R5; data-model.md "State transitions"):
  `src/Shop.Api/Payments/ExpiryJob.cs` (hosted, every 15 s) expiring only when
  `deadline < now` and `deadline < seen_through_time` of the order's address, returning stock in
  the same step (PostgreSQL conditional update; Redis
  `src/Shop.Api/Storage/Redis/Scripts/expire_order.lua`), clearing the waiting key;
  `GET /internal/wallets/{address}/waiting`. Tests: 3 concurrent jobs expire once and return stock
  once; no expiry while `seen_through_time` is behind the deadline; a deleted item gets nothing
  back; restart mid-way.
- [ ] T011 [P] [US1] Storefront (contracts/shop-api.md "Angular"): `src/Shop.Web/src/app/checkout/`
  connect MetaMask and send `payingAccount`, link to the waiting order on 409;
  `src/Shop.Web/src/app/order/` order page with total, address, time left, status polling every
  5 s, "Pay with MetaMask" (switch chain, same-account check, mUSDT and ETH balance checks,
  `transfer`), "payment sent, confirming" from `localStorage`, "Get test mUSDT" (`mint`);
  `src/Shop.Web/src/app/shared/eip1193.ts` with the three selectors. Tests with a fake EIP-1193
  provider for each scenario of US1 (spec acceptance 1–6).
- [ ] T012 [P] [US4] Admin view (contracts/shop-api.md "Admin"): order status, deadline and
  payment link in `GET /api/admin/orders`; `GET /api/admin/payments?matched=false`; Angular admin
  orders page status column and "Payments not matched" list in `src/Shop.Web/src/app/admin/`.
  Tests: API and component.
- [ ] T013 [US1] Release shop: PR(s) with CI green → **ask before merge** → **ask before tag**
  `v0.3.0` → image `ghcr.io/nikola031ue/shop:0.3.0`; record tag-to-image time here.

**Checkpoint**: shop 0.3.0 image published. It is not rolled out before T024 (together with
operator 0.4.0): under operator 0.3.1 it would refuse every checkout (`paymentsUnavailable`) and
open no port 8081. Commit (spec repo).

---

## Phase 2: Operator 0.4.0 + chart `shop-operator` 0.5.0 (US1, US3) — repos `shop-operator`, `helm-charts`

**Goal**: one watcher delivers each shop's confirmed transfers; shops get their internal token,
port and settings; Ready means no old copies; old Wallets wait for their orders; the chart ships
the delete guard and the egress policy.
**Independent test**: envtest + Anvil with a fake shop server: transfers delivered once per range
after 6 blocks, catch-up pages ≤10 blocks, cursor advances only on 204; Wallet deletion held until
waiting = 0; readiness false while old pods run (quickstart Scenario 1, operator part).

- [ ] T014 [P] [US3] D-5 readiness (contracts/operator.md "Shop readiness", research R10) in
  `internal/controller/shop_controller.go` `availableReplicas`: rollout in progress unless
  `replicas == updatedReplicas` and no terminating replicas besides the existing checks. Test:
  envtest with a Deployment status showing an extra old replica → not Ready (reproduces 004 T025 E3).
- [ ] T015 [US1] Shop wiring (contracts/operator.md "Shop app wiring"; data-model.md "Derived
  Secret" `<shop>-internal.token`): generate the Secret once in `internal/resources/secrets.go`;
  env `TOKEN_ADDRESS`, `CHAIN_ID`, `EXPLORER_TX_URL`, `ORDER_PAYMENT_MINUTES`, `INTERNAL_TOKEN`
  in `internal/resources/resources.go`; Service port `internal` 8081 and NetworkPolicy ingress
  TCP 8081 from the operator's pods in `internal/resources/routing.go`; flags for `payments.*`
  (data-model.md "Platform settings and Secrets") in `internal/controller/config.go` and
  `cmd/main.go`. Tests: resources and envtest (Secret never regenerated, env present, Ingress
  still on `http`).
- [ ] T016 [US1] Watcher (research R1–R3; contracts/operator.md "Chain watcher"):
  `internal/chain/watcher.go` leader-elected Runnable every `pollSeconds`; confirmed head =
  head − `confirmations`; one `eth_getLogs` per ≤ `logsBlockRange` range with the Wallet topic
  OR-filter; Wallet status `payments.scannedThroughBlock`/`scannedThroughTime` (data-model.md
  "Wallet status") initialised at the confirmed head; condition `PaymentsWatching` with reasons
  `RPCCredentialsMissing`, `TokenNotConfigured`, `ProviderError`; RPC URL read from the Secret,
  only its host ever logged. Add the status fields to `api/v1alpha1/wallet_types.go` and
  regenerate the CRD. Tests on Anvil: confirmations respected, 1 h catch-up in ≤10-block pages,
  no Secret/no token address → idle with the right reason.
- [ ] T017 [US1] Delivery (contracts/shop-internal-api.md; contracts/operator.md):
  `internal/chain/deliver.go` posts per Wallet (label `shophub.io/shop`) with the bearer token,
  heartbeat when no transfers, 5 s timeout, cursor patched only after 204, `ShopUnreachable`
  without blocking other shops, backoff 12 s → 2 min on provider errors. Tests with a fake shop
  server: failure holds only that Wallet's cursor; redelivery after recovery; no transfer lost or
  delivered to the wrong shop.
- [ ] T018 [US3] D-1 Wallet release (research R9; contracts/operator.md "Wallet deletion") in
  `internal/controller/wallet_controller.go`: finalizer `shophub.io/in-use-protection` removed
  only when no Shop references it and (owning Shop gone or being deleted with
  `shophub.io/delete-requested`, or fully rolled out per T014 and `/internal/wallets/{address}/waiting`
  = 0); otherwise `Ready=False` `WaitingForOrders`, requeue 60 s. Tests: envtest + fake shop
  (held while waiting = 2, released at 0; released at once when the Shop is deleted through
  ShopHub; held while an old pod runs).
- [ ] T019 [US1] Release operator: PR(s) with CI green → **ask before merge** → **ask before tag**
  `v0.4.0` → image; `hack/sync-chart.sh` output for the chart.
- [ ] T020 [US3] Chart `shop-operator` 0.5.0 in helm-charts (contracts/operator.md "Shop deletion
  guard", "Chart"): `charts/shop-operator/values.yaml` `payments.*` and `shopDeleteGuard.enabled`;
  `payments.rpcSecretName` as optional `secretKeyRef` in `templates/deployment.yaml`; new
  `templates/admission-policy.yaml` (ValidatingAdmissionPolicy + Binding, research R11) and
  `templates/networkpolicy.yaml` (egress, research R13); synced CRDs; appVersion `0.4.0`, shop app
  image default `0.3.0`; pin the kind node image in `.github/workflows/charts.yaml`. The policy
  carries the `matchConditions` exemption for the namespace controller, the garbage collector and
  `system:kube-controller-manager` (contracts/operator.md). The `install` job asserts: a Shop
  `DELETE` without the annotation is denied; with it, allowed; and deleting a namespace that
  still holds an unannotated Shop finishes within 60 s (namespace and Shop gone). PR → **ask
  before merge** → published.

**Checkpoint**: operator 0.4.0 and chart 0.5.0 published. Commit (spec repo).

---

## Phase 3: ShopHub 0.3.0 (US3) — repos `shophub`, `helm-charts`

**Goal**: ShopHub never shows a stale Ready, and its deletes carry the annotation the guard needs.
**Independent test**: k3s with the guard applied: ShopHub's delete passes, a bare delete is
denied; a status with an older `observedGeneration` maps to "being prepared".

- [ ] T021 [US3] D-6 and D-4 in `src/ShopHub.Api/Shops/KubeGateway.cs` (research R11, R12;
  data-model.md "ShopHub"): `GetShopStatusAsync` not ready (reason text "Applying your change." in
  `src/ShopHub.Api/Shops/ReasonTexts.cs`) while `status.observedGeneration < metadata.generation`;
  `DeleteShopAsync` patches `shophub.io/delete-requested: "true"` on the Shop, then deletes it.
  Tests: unit for the stale status (API PATCH answer shows "being prepared"); k3s
  `tests/ShopHub.Api.Tests/Integration/K3s/` applying the guard manifest: ShopHub's delete works,
  a delete without the annotation is denied, request recording still shows only by-name calls.
  The guard is applied with the same binding as production (whole shops namespace), so the
  existing direct delete of a stray Shop in `ReconcilerTests.cs` (line 249) first patches
  `shophub.io/delete-requested: "true"` onto it.
- [ ] T022 [US3] Release ShopHub: PR with CI green → **ask before merge** → **ask before tag**
  `v0.3.0` → image; chart `shophub` 0.2.2 (appVersion `0.3.0`) PR → **ask before merge** →
  published.

**Checkpoint**: ShopHub 0.3.0 and chart 0.2.2 published. Commit (spec repo).

---

## Phase 4: Rollout + live Sepolia check — repo `kube-state`, docker-desktop

**Goal**: the platform runs 005 and the quickstart passes on Sepolia.

- [ ] T023 kube-state PR A: `clusters/local/shophub/{helm.yaml,values.yaml}` chart 0.2.2, image
  `0.3.0` → **ask before merge** → `./bring-up.sh` (ShopHub first, plan "Rollout order"); check
  panel delete still works with the old operator.
- [ ] T024 kube-state PR B: `clusters/local/shop-operator/{helm.yaml,values.yaml}` chart 0.5.0,
  `payments.tokenAddress` from T004, shop app image `0.3.0` → **ask before merge** →
  `./bring-up.sh`; quickstart Scenario 2 (Wallets `PaymentsWatching=True`, cursors follow head − 6,
  logs show the provider host only).
- [ ] T025 Live payments with the maintainer in the browser (M4 wallets): quickstart Scenarios 3
  (pay, FR-005a refusal), 4 (wrong amount, wrong account, duplicate) and 5 (expiry on a `high`
  shop, then a late payment); record SC-001, SC-002, SC-003 timings here.
- [ ] T026 Live paths: quickstart Scenarios 6 (operator scaled to 0 for 15 min, hand change
  recorded), 7 (wallet change with a waiting order: D-6 within 5 s, D-5, D-1 hold and release),
  8 (`kubectl delete shop` denied by the guard, hand attempt recorded) and 9 (delete through
  ShopHub, gone within 2 min); record SC-004 to SC-007.
- [ ] T027 Secret sweep (quickstart Scenario 10, SC-008) including the RPC URL and every
  `<shop>-internal` token, run before any shop is deleted; clean up test shops through the panel.

**Checkpoint**: all scenarios recorded. Commit (spec repo).

---

## Phase 5: Polish

- [ ] T028 [P] READMEs via PRs (**ask before each merge**): shop (payments, internal port,
  settings), shop-operator (watcher settings, token and Remix deploy, RPC Secret, delete guard),
  kube-state (RPC Secret as a hand-made step next to the Discord token; `payments.tokenAddress`).
- [ ] T029 Self-review of all 005 PRs (findings with file and issue); plan.md "Deferred
  follow-ups" for anything left; spec.md limits checked against T025–T027; status Implemented
  after the maintainer's OK.

---

## Dependencies & Execution Order

### Phase Dependencies

- Phase 0 order: T001 → T002 (Secret needs the endpoint); T003 merged → T004 (Remix deploys the
  merged source); T005 any time before Phase 4.
- Phase 0 → everything (T003 token for operator tests; T004 address for T024; T002 for T024).
- Phase 1 (shop) and Phase 2 (operator) can be developed in parallel once T003 is merged; the
  operator's delivery tests use a fake shop, the shop's tests call `/internal/chain` directly.
- Phase 2 T020 needs T013 (shop image default) and T019.
- Phase 3 is independent of Phases 1–2 in code; its rollout (T023) must precede T024.
- Phase 4 needs all releases; Phase 5 last.

### User Story Dependencies

- US1 (pay) needs T006–T009, T011 (shop) and T015–T017 (operator).
- US2 (expiry) needs T010 and the watcher's heartbeat (T017).
- US3 (paths) needs T014, T018, T020, T021; it is checked live in T026.
- US4 (admin) needs T012 and US1's data.

### Within Each Phase

Schema → store → endpoints → background jobs → UI → release.

### Parallel Opportunities

- T011 and T012 (Angular, different folders) while T009/T010 are in review.
- T014 alongside T015–T017 (different files).
- Phase 3 (shophub) alongside Phases 1–2 (different repos).
- T028 READMEs in three repos at once.

## Parallel Example: Phases 1 and 2

```text
shop:          T006 → T007 → T008 → T009 → T010 ; T011 ∥ T012 → T013
shop-operator: T014 ∥ (T015 → T016 → T017 → T018) → T019 → T020
shophub:       T021 → T022
```

## Implementation Strategy

### MVP First (User Story 1)

Phases 0–2 with US1 tasks give a payable order end to end on a local chain (shop + operator +
token). US2 (expiry) completes the P1 pair; together they are the MVP for Requirement 2.4.

### Incremental Delivery

1. Token on Anvil (Phase 0) → 2. shop 0.3.0 (inert without the operator) → 3. operator 0.4.0 +
chart 0.5.0 → 4. ShopHub 0.3.0 → 5. rollout ShopHub first, then the operator → 6. live check.

### Solo Developer

One phase at a time with a commit at each end; Phases 1 and 3 can interleave while PRs wait for
approval.

## Notes

- [MANUAL] tasks: the agent never handles the provider URL or keys; it checks results only (key
  present, address recorded).
- Every merge, tag, release, chart publish and kube-state merge asks the maintainer first.
- Hand cluster changes only where recorded (T026: operator scaled to 0, a hand delete attempt).
