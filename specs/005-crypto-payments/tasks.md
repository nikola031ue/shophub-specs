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

- [X] T001 [MANUAL] **M1 provider account**: maintainer creates a free Sepolia endpoint at Infura
  or Alchemy (quickstart M1). Done when the maintainer confirms an HTTPS endpoint URL with key
  exists; nothing is recorded anywhere.
- [X] T002 [MANUAL] **M2 RPC Secret**: maintainer runs
  `kubectl create secret generic shop-operator-rpc -n shop-operator-system --from-literal=url="$SEPOLIA_RPC_URL"`
  on docker-desktop (quickstart M2; data-model.md "Platform settings and Secrets"
  `payments.rpcSecretName`). Agent checks only that key `url` is non-empty (prints its length).

  Done (maintainer, 2026-11-27). Checked on docker-desktop without printing it: Secret
  `shop-operator-rpc` has only key `url`, 58 characters, starts with `https://`, no surrounding
  whitespace. M1 is done by implication (the endpoint exists).
- [X] T003 Token source and tests in shop-operator (contracts/musdt.md, research R8):
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

  Progress (2026-11-27):
  - **shop-operator PR #9** (branch `005-token`), CI green.
  - **Token:** `contracts/MockUSDT.sol` (OpenZeppelin 5.1.0 pinned in `package.json`/lockfile;
    `MAX_MINT`, `MintTooLarge(requested, max)`); `contracts/build.sh` with `ethereum/solc:0.8.28`,
    optimizer 200, EVM cancun; committed `build/MockUSDT.{abi,bin}`. The CI rebuild on Linux is
    byte-identical to the build committed from Windows.
  - **Tests:** `internal/chain/{anvil,token}_test.go` on Anvil `ghcr.io/foundry-rs/foundry:v1.0.0`
    via testcontainers-go v0.37.0 (Anvil default accounts). They cover metadata, the mint cap,
    `transfer` and the `Transfer` log, plus the hard-coded selectors and topic.
  - **CI checks:** `hack/check-no-chain-hosts.sh` matches host names, provider domains and the
    chain id, not the word "Sepolia" (CRD descriptions use it). A planted provider URL is caught.
  - **Mutation:** cap raised to 2,000 → the cap test fails.
  - **Lint:** `golangci-lint` 0 issues (run in the Linux dev container; the local `make` and lint
    binaries are Linux-only). The other operator test packages pass.
  - **Fix on the way:** the first CI run failed with "receipt: not found". Anvil's receipt can lag
    the send on a slow runner, so the helper now waits for it (30 s, 100 ms steps).
  - Merged as shop-operator `f54cecf` (approved, 2026-11-27).
- [X] T004 [MANUAL] **M3 Remix deploy** (only after T003 is merged, so the deployed source is the
  reviewed, tested one): maintainer deploys `contracts/MockUSDT.sol` from `main` on Sepolia
  from their own MetaMask with the settings in `contracts/README.md`, and verifies the source on
  the explorer (quickstart M3). Agent records the address in `contracts/README.md` (PR, ask before
  merge) and in this file; it goes into kube-state in T024.
  Bytecode check (maintainer, 2026-11-27): Remix's output may differ only in the trailing CBOR
  metadata (it hashes source paths); compare with that stripped (the last 2 bytes give its length),
  and only the code before it must match `contracts/build/MockUSDT.bin`.
  - Compare script first proven both ways: a one-byte change in the code gives DIFFERENT; a
    one-byte change inside the metadata only gives MATCH.
  - Remix compile (before deploy, 2026-11-27): code before the metadata **matches** the
    committed build byte for byte (2,373 bytes); only the 51-byte CBOR metadata differs, as
    expected. Its trailer says solc 0.8.28.
  - **Deployed** (maintainer, 2026-11-27): **`0xdF8d5dEb736EBC79f7e99C4c90975d4b8f8d80E2`** on
    Sepolia (EIP-55 checksum valid).
  - **Checked read-only** through a public keyless endpoint (the RPC Secret wasn't used):
    - chain id `0xaa36a7`;
    - the runtime code, with the metadata stripped, is the runtime part of the committed build;
    - name "Mock USDT", symbol mUSDT, 6 decimals, `MAX_MINT` 1,000,000,000, total supply 0;
    - Sourcify: exact match for creation and runtime (verified 10:54:46 UTC).
  - **README:** address recorded in `contracts/README.md` in shop-operator PR #10 (waiting for
    merge approval). It goes into kube-state in T024.
- [ ] T005 [MANUAL] **M4 tester wallets**: maintainer prepares MetaMask accounts A and B with
  Sepolia ETH from a public faucet (quickstart M4). Needed by Phase 4 only.

  Progress: A = `0xE3C0…A2Cf`, B = `0x7294…2285` (maintainer, 2026-11-27); B is being funded with
  Sepolia ETH.

**Checkpoint**: token on Anvil in CI; address known; Secret present. Commit (spec repo).
Phase 0 reached 2026-11-27: T001–T004 done. T005: accounts A and B named, B still being funded
(needed in Phase 4 only).

---

## Phase 1: Shop app 0.3.0 (US1, US2, US4) — repo `shop`

**Goal**: orders carry a paying account and deadline; the shop receives transfers on its internal
port, matches or records them, expires orders behind the chain, and shows payment in the
storefront and admin.
**Independent test**: with the operator replaced by test calls to `/internal/chain`, an exact
transfer marks an order Paid once, every wrong one is recorded with its reason, expiry returns
stock once (quickstart Scenario 1, shop part).

- [X] T006 [US1] Settings and schema: `src/Shop.Api/Settings.cs` reads `TOKEN_ADDRESS`, `CHAIN_ID`,
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
- [X] T007 [US1] Checkout and order lookup (contracts/shop-api.md "Public"): `POST /api/orders`
  takes `payingAccount` (EIP-55, data-model.md `Order.paying_account`), stores
  `total_base_units = cents × 10^4` and `deadline`; the one-waiting-order rule (data-model.md
  "Uniqueness (FR-005a)": PostgreSQL partial unique index, Redis NX key in
  `src/Shop.Api/Storage/Redis/Scripts/place_order.lua`) answering 409 `waitingOrder` with the
  reference; 409 `paymentsUnavailable`; `GET /api/orders/{reference}`; `GET /api/shop` `payments`
  block. Files: `src/Shop.Api/Endpoints/Public.cs`, `src/Shop.Api/Storage/IShopStore.cs`, both
  stores. Tests on both stores, including two concurrent checkouts with one account (one wins).
- [X] T008 [US1] Internal endpoint (contracts/shop-internal-api.md): second Kestrel endpoint on
  8081 in `src/Shop.Api/Program.cs`, `src/Shop.Api/Endpoints/Internal.cs` with constant-time bearer
  check against `INTERNAL_TOKEN`; `/internal/*` answers 404 on 8080. Tests: 401 without/with wrong
  token, 404 on the public port, 400 for a foreign address.
- [X] T009 [US1] Matching (research R6; contracts/shop-internal-api.md "POST /internal/chain"):
  `src/Shop.Api/Payments/Matching.cs`; PostgreSQL in one transaction per transfer, Redis
  `src/Shop.Api/Storage/Redis/Scripts/record_transfer.lua`; idempotent on (`tx`, `logIndex`);
  sets data-model.md `Payment.order_reference` on a match, otherwise `Payment.unmatched_reason`
  (`amount`, `account`, `late`, `duplicate`, `unknown`); clears the waiting key on Paid; `chain_progress` only moves forward
  (data-model.md `Chain progress.seen_through_block`/`seen_through_time`); 204 only when all is
  stored. Tests on both stores: each reason, redelivery of the same range, two shop copies
  receiving the same transfer at once (Paid once).
- [X] T010 [US2] Expiry and waiting count (research R5; data-model.md "State transitions"):
  `src/Shop.Api/Payments/ExpiryJob.cs` (hosted, every 15 s) expiring only when
  `deadline < now` and `deadline < seen_through_time` of the order's address, returning stock in
  the same step (PostgreSQL conditional update; Redis
  `src/Shop.Api/Storage/Redis/Scripts/expire_order.lua`), clearing the waiting key;
  `GET /internal/wallets/{address}/waiting`. Tests: 3 concurrent jobs expire once and return stock
  once; no expiry while `seen_through_time` is behind the deadline; a deleted item gets nothing
  back; restart mid-way.
- [X] T011 [P] [US1] Storefront (contracts/shop-api.md "Angular"): `src/Shop.Web/src/app/checkout/`
  connect MetaMask and send `payingAccount`, link to the waiting order on 409;
  `src/Shop.Web/src/app/order/` order page with total, address, time left, status polling every
  5 s, "Pay with MetaMask" (switch chain, same-account check, mUSDT and ETH balance checks,
  `transfer`), "payment sent, confirming" from `localStorage`, "Get test mUSDT" (`mint`);
  `src/Shop.Web/src/app/shared/eip1193.ts` with the three selectors. Tests with a fake EIP-1193
  provider for each scenario of US1 (spec acceptance 1–6).
- [X] T012 [P] [US4] Admin view (contracts/shop-api.md "Admin"): order status, deadline and
  payment link in `GET /api/admin/orders`; `GET /api/admin/payments?matched=false`; Angular admin
  orders page status column and "Payments not matched" list in `src/Shop.Web/src/app/admin/`.
  Tests: API and component.
- [X] T013 [US1] Release shop: PR(s) with CI green → **ask before merge** → **ask before tag**
  `v0.3.0` → image `ghcr.io/nikola031ue/shop:0.3.0`; record tag-to-image time here.

  Progress T006–T012 (2026-11-27):
  - **shop PR #9** (branch `005-payments`): one PR for T006–T012, CI green.
  - **Store:** PostgreSQL script `004_payments.sql`; Redis schema 2 (an order's JSON is never
    rewritten, its payment state is a hash `shop:order-state:{ref}`, so totals are never
    re-encoded by Lua).
  - **Rules:**
    - one waiting order per account: partial unique index or NX key; the concurrent-checkout
      race is turned into `waitingOrder`;
    - matching in `PaymentMatching.Decide` (C#) and `record_transfer.lua`, both held to the
      same contract tests;
    - expiry only behind chain progress, stock back in the same step.
  - **Internal API:** Kestrel listener on `INTERNAL_PORT` (8081) only with `INTERNAL_TOKEN`. Its
    connections are marked, so a forged `Host` header doesn't help; constant-time bearer check;
    the listener serves only `/internal`; `/internal/*` is 404 on the public port.
  - **Settings:** all optional, so 0.3.0 also starts under operator 0.3.1 (payments unavailable).
  - **Angular:**
    - EIP-1193 without a library (`shared/eip1193.ts`, `store/wallet.ts`);
    - checkout "Connect MetaMask";
    - new `store/order-page.ts` (replaces `order-placed.ts`);
    - admin "Payments not matched".
  - **Tests:** 281 .NET (both stores; real Kestrel ports for the internal API; upgrade from
    0.2.0 on both stores; settings; EIP-55) and 35 Angular. `dotnet format` and `ng lint` are
    clean.
  - **Mutation checks**, each caught by a failing test:
    - no chain-progress guard (both stores);
    - no Redis idempotence;
    - no unique index;
    - internal filter without the connection mark;
    - no listener scope.
  - **Contract change:** status values stay spec 003's PascalCase (`PendingPayment`, `Paid`,
    `Expired`), as now written in contracts/shop-api.md.
  - **Test-only fixes on the way:** a reference-collision test now uses another account (the
    waiting check answers first); `/api/orders/../../etc` never reaches the endpoint (HttpClient
    normalises it), so a reference with a character outside the alphabet replaced it.
  - **Image:** the shop listens on 80 (not 8080); the image now also declares `EXPOSE 8081`.
  - Merged as shop `4ea761e` (approved, 2026-11-27); main CI green.
  - **T013 release:** `v0.3.0` tagged on `4ea761e` (approved). Tag pushed 11:42:01 UTC; the
    `release` job finished 11:44:36, so **2 min 35 s** to `ghcr.io/nikola031ue/shop:0.3.0`; the
    image can be pulled anonymously (HTTP 200).

**Checkpoint**: shop 0.3.0 image published (reached 2026-11-27). It is not rolled out before T024 (together with
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

- [X] T014 [P] [US3] D-5 readiness (contracts/operator.md "Shop readiness", research R10) in
  `internal/controller/shop_controller.go` `availableReplicas`: rollout in progress unless
  `replicas == updatedReplicas` and no terminating replicas besides the existing checks. Test:
  envtest with a Deployment status showing an extra old replica → not Ready (reproduces 004 T025 E3).
- [X] T015 [US1] Shop wiring (contracts/operator.md "Shop app wiring"; data-model.md "Derived
  Secret" `<shop>-internal.token`): generate the Secret once in `internal/resources/secrets.go`;
  env `TOKEN_ADDRESS`, `CHAIN_ID`, `EXPLORER_TX_URL`, `ORDER_PAYMENT_MINUTES`, `INTERNAL_TOKEN`
  in `internal/resources/resources.go`; Service port `internal` 8081 and NetworkPolicy ingress
  TCP 8081 from the operator's pods in `internal/resources/routing.go`; flags for `payments.*`
  (data-model.md "Platform settings and Secrets") in `internal/controller/config.go` and
  `cmd/main.go`. Tests: resources and envtest (Secret never regenerated, env present, Ingress
  still on `http`).
- [X] T016 [US1] Watcher (research R1–R3; contracts/operator.md "Chain watcher"):
  `internal/chain/watcher.go` leader-elected Runnable every `pollSeconds`; confirmed head =
  head − `confirmations`; one `eth_getLogs` per ≤ `logsBlockRange` range with the Wallet topic
  OR-filter; Wallet status `payments.scannedThroughBlock`/`scannedThroughTime` (data-model.md
  "Wallet status") initialised at the confirmed head; condition `PaymentsWatching` with reasons
  `RPCCredentialsMissing`, `TokenNotConfigured`, `ProviderError`; RPC URL read from the Secret,
  only its host ever logged. Add the status fields to `api/v1alpha1/wallet_types.go` and
  regenerate the CRD. Tests on Anvil: confirmations respected, 1 h catch-up in ≤10-block pages,
  no Secret/no token address → idle with the right reason.
- [X] T017 [US1] Delivery (contracts/shop-internal-api.md; contracts/operator.md):
  `internal/chain/deliver.go` posts per Wallet (label `shophub.io/shop`) with the bearer token,
  heartbeat when no transfers, 5 s timeout, cursor patched only after 204, `ShopUnreachable`
  without blocking other shops, backoff 12 s → 2 min on provider errors. Tests with a fake shop
  server: failure holds only that Wallet's cursor; redelivery after recovery; no transfer lost or
  delivered to the wrong shop.
- [X] T018 [US3] D-1 Wallet release (research R9; contracts/operator.md "Wallet deletion") in
  `internal/controller/wallet_controller.go`: finalizer `shophub.io/in-use-protection` removed
  only when no Shop references it and (owning Shop gone or being deleted with
  `shophub.io/delete-requested`, or fully rolled out per T014 and `/internal/wallets/{address}/waiting`
  = 0); otherwise `Ready=False` `WaitingForOrders`, requeue 60 s. Tests: envtest + fake shop
  (held while waiting = 2, released at 0; released at once when the Shop is deleted through
  ShopHub; held while an old pod runs).
- [X] T019 [US1] Release operator: PR(s) with CI green → **ask before merge** → **ask before tag**
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

Progress T014–T018 (2026-11-27):
  - **shop-operator PR #11** (branch `005-operator`): one PR for T014–T018, CI green.
  - **T014 D-5:** `RolloutInProgress` (also `replicas == updatedReplicas`, no terminating
    replicas). Unit table test; envtest: Progressing while an extra old copy runs, then Ready.
  - **T015 wiring:**
    - `<shop>-internal` (32-byte token, created once, owned by the Shop);
    - the payment env, with `INTERNAL_TOKEN` as a `secretKeyRef`;
    - port `internal` 8081 on the container and Service;
    - a NetworkPolicy rule for 8081 from the operator's pods only (`--operator-pod-selector`).
  - **T016–T017 watcher:** `internal/chain/watcher.go` with delivery through
    `internal/shopapi` (its own package, to avoid an import cycle with the Wallet controller).
    Anvil tests:
    - delivered once, only after the confirmations;
    - catch-up in ≤ N-block ranges;
    - the cursor held while the shop fails;
    - missing credentials reported;
    - the URL's key never in status.
  - **T018 D-1:** the Wallet finalizer is held (`WaitingForOrders`, with the reason) while the
    shop rolls out or the shop reports waiting orders; released at once when the shop is
    deleted.
  - **Stall visibility (maintainer, 2026-11-27):**
    - the watcher writes only `status.payments` (cursor, `lastError`);
    - the Wallet controller derives `PaymentsWatching` every minute and reports `Stalled` when
      the read position is older than `--payments-stall-after` (5 m);
    - a watcher loop that stopped is visible, because the condition doesn't depend on it writing.
  - **Checks:**
    - envtest 87 specs and the Go unit tests pass; `golangci-lint` 0 issues;
      `check-no-chain-hosts` clean (it caught Sepolia values in a new resource test, now
      neutral);
    - mutation checks, each caught: no confirmations; cursor advanced on a shop failure; no
      paging; D-1 without the rollout check; D-1 ignoring the waiting count; no `Stalled`.
  - **Test-only change:** `markDeploymentAvailable` lost its count parameter (always 2; lint).

  - Merged as shop-operator `4eeb4bc` (approved, 2026-11-27); main CI green.
  - **T019 release:** `v0.4.0` tagged on `4eeb4bc` (approved). Tag pushed 12:31:23 UTC; the
    `release` job finished 12:35:24, so **4 min 01 s** to `ghcr.io/nikola031ue/shop-operator:0.4.0`;
    the image can be pulled anonymously.
  - **T020:** helm-charts PR #11 (branch `shop-operator-0.5.0`), CI green, waiting for merge
    approval.
    - Chart 0.5.0 (appVersion 0.4.0): `payments.*` values as flags, `--operator-pod-selector`
      from the chart's selector labels, `templates/admission-policy.yaml` (the guard, with the
      controller exemption), `templates/networkpolicy.yaml` (egress), synced CRDs/RBAC,
      `shopImage` default shop 0.3.0.
    - CI pins kind `kindest/node:v1.32.2`. The `install` log shows: egress policy present; an
      unannotated delete "denied request: Shops are deleted through ShopHub…"; annotated
      delete ok; `namespace "shops" deleted` with an unannotated Shop still in it.

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

  **Hard rule (maintainer, 2026-11-27):** kube-state must not move the operator chart to 0.5.0
  until ShopHub 0.3.0 is running. Chart 0.5.0's delete guard refuses a Shop delete without
  `shophub.io/delete-requested`, which older ShopHub doesn't write, so panel deletes would fail.
  Enforced twice in this task:
  - a **kube-state CI check** (validate workflow, with a bats test): fail when
    `clusters/local/shop-operator/helm.yaml` pins chart ≥ 0.5.0 while
    `clusters/local/shophub/values.yaml` has `image.tag` < 0.3.0;
  - a **live check before `./bring-up.sh`**: the running ShopHub Deployment's image is
    `ghcr.io/nikola031ue/shophub:0.3.0` and its rollout is complete; otherwise stop.
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
