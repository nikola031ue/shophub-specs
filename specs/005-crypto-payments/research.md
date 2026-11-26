# Research: Crypto Payments (005)

Decisions for the plan. Each: **Decision**, **Rationale**, **Alternatives considered**.

## R1 — One chain watcher, in the shop-operator

- **Decision**: A single watcher runs in the shop-operator as a leader-elected manager Runnable
  (the chart runs 1 replica with `--leader-elect`, so exactly one watcher is active). It reads the
  chain for every Wallet in `shops` and hands what it finds to the owning shop. The Wallet
  controller keeps each Wallet's watch state in its status.
- **Rationale**: Wallet exists to "create an account on the chain that receives payments"
  (Requirement 3.1). One reader keeps provider usage within a free plan (one query per block
  range for all Wallets), and no two readers race to mark the same order paid. The shop app
  copies (2–3 per shop) stay free of chain code and provider credentials.
- **Alternatives**: a watcher in every shop copy (N×replicas readers on a free plan, races
  between copies, the provider Secret in every shop namespace); one in ShopHub (it would need to
  write into shops' databases or call every shop, and ShopHub isn't the Wallet's owner).

## R2 — Reading transfers: one paged log query for all Wallets

- **Decision**: Every 12 s: read the head block; the confirmed head is `head − confirmations`
  (default 6). From the lowest Wallet cursor + 1 up to the confirmed head, page through ranges of
  at most `logsBlockRange` blocks (default 10, a setting) with one `eth_getLogs` per range:
  `address = mUSDT`, `topics[0] = Transfer(address,address,uint256)`,
  `topics[2] ∈ {padded Wallet addresses}`. Each log gives (tx hash, log index, block, from, to,
  value); block times come from the block headers of blocks that have logs, plus the last block
  of each range.
- **Rationale**: free provider plans cap the block span of one log query (maintainer note;
  some free plans allow as little as 10 blocks), so catch-up after 1 h (≈ 300 blocks) is about
  30 calls, well inside free quotas. One OR-filter on `topics[2]` covers all Wallets at once.
  Reading only up to the confirmed head means nothing is delivered before 6 blocks, so no
  delivered transfer can be undone by a short reorganisation.
- **Alternatives**: subscriptions (`eth_subscribe`) need websockets and still need catch-up after
  a gap; per-Wallet queries multiply calls; reading to the head and retracting on reorg is more
  code for no user gain.

## R3 — Cursor: per-Wallet status, start at creation

- **Decision**: `Wallet.status.payments.scannedThroughBlock` (and its block time) is the last
  confirmed block whose transfers the owning shop has **accepted**. A new Wallet starts at the
  confirmed head when first seen. The watcher advances a Wallet's cursor only after the shop
  answered 2xx for that range.
- **Rationale**: no new storage; survives operator restarts; a shop that is down simply holds
  its own cursor back (at-least-once delivery; the shop is idempotent on (tx hash, log index)).
  The global query starts at the lowest cursor, so one lagging shop re-reads a few ranges, not
  the others' state.
- **Alternatives**: a ConfigMap or database for the cursor (more state to own); exactly-once
  delivery (not needed with idempotent receipt).

## R4 — Handing transfers to the shop: internal endpoint, own port, per-shop token

- **Decision**: The operator posts to the shop's in-cluster Service on a second port, `internal`
  (8081), which the Ingress never routes: `POST /internal/chain` with the transfers for that
  shop's address and `seenThrough` (block, time). Auth: a bearer token from an operator-generated
  Secret `<shop>-internal` (like `<shop>-admin`), injected into the shop as `INTERNAL_TOKEN`. The
  shop's NetworkPolicy admits port 8081 only from the operator's pods. The shop matches,
  records and expires; the operator never interprets orders.
- **Rationale**: order matching needs the shop's database (PostgreSQL or Redis, with its own
  atomic scripts), so it belongs in the shop; one DB transaction per transfer makes "paid once"
  hold across shop copies. Separate port + NetworkPolicy + token is three layers; any one leak
  alone doesn't open it to the internet.
- **Alternatives**: operator writing into shop databases (two storage engines, crosses the
  derived-resource boundary); Wallet status as a mailbox the shop reads (shops have no Kubernetes
  API access, by design).

## R5 — Expiry only behind the chain cursor (FR-011)

- **Decision**: Each shop copy runs an expiry job every 15 s: in one atomic step per order, set
  `Expired` and return stock **only if** `status = PendingPayment AND deadline < now AND
  deadline < seenThrough.time`, where `seenThrough` is the last value the operator delivered for
  that address. The operator delivers `seenThrough` on every pass (empty `transfers` is a
  heartbeat), so an idle shop still advances.
- **Rationale**: an order must not expire while a payment for it might sit in blocks not yet
  read; with the guard, an outage pauses expiry instead of wrongly expiring paid orders. The
  conditional state change is what makes it exactly once with several copies.
- **Alternatives**: expiry in the operator (doesn't know orders); expiry by wall clock alone
  (breaks FR-011).

## R6 — Matching, the one-waiting-order rule and unmatched transfers

- **Decision**: In the shop, per delivered transfer (idempotent on tx hash + log index): find the
  order with `paying_account = from AND wallet_address = to AND status = PendingPayment AND
  total_base_units = value AND placed_block_time <= transfer block time`; if found and the
  transfer's block time is ≤ its deadline, mark it `Paid` with the payment; otherwise record the
  transfer as unmatched with a reason (`amount`, `account`, `late`, `duplicate`, `unknown`).
  FR-005a: at most one `PendingPayment` order per (paying account) in a shop — PostgreSQL: a
  partial unique index; Redis: a key per waiting account set in `place_order.lua` and cleared on
  paid/expired.
- **Rationale**: the uniqueness rule makes the match unambiguous (clarification Q1); a late
  transfer is recorded only (Q2). Block time, not arrival time, decides lateness, so an outage
  never turns an on-time payment into a late one.
- **Alternatives**: oldest-match tie-break (unneeded with the rule).

## R7 — Amounts: 6 decimals

- **Decision**: mUSDT has 6 decimals, like USDT. A price of `x.yz` (shop stores cents) is
  `cents × 10^4` base units. The order stores `total_base_units` at placement.
- **Rationale**: one exact integer comparison; no floating point.

## R8 — The token: OpenZeppelin ERC20 with a capped open mint; source in shop-operator

- **Decision**: `MockUSDT` (name "Mock USDT", symbol `mUSDT`, 6 decimals) on OpenZeppelin
  Contracts ERC20, plus `mint(uint256 amount)` that mints to the caller and reverts above
  1,000 mUSDT per call. Source in `shop-operator/contracts/MockUSDT.sol` with the compiled ABI and
  bytecode committed; CI recompiles with a pinned `solc` and fails on any difference. Tests deploy
  the bytecode on Anvil (testcontainers-go) and check mint cap, transfer and the Transfer event.
  **Manual**: the maintainer deploys it once on Sepolia with Remix and their own MetaMask, and
  records the address in kube-state.
- **Rationale**: the operator is the only component that reads the token, so its tests already
  need it on Anvil; no new repository. Committed artifacts + CI recompile make the deployed
  bytecode reproducible from the repo.
- **Alternatives**: a separate `contracts` repo (one more repo for ~30 lines); Foundry in CI (a
  second toolchain; Go tests on Anvil suffice); CI deploys (needs a funded key in CI and contacts
  Sepolia — forbidden).

## R9 — D-1: the Wallet finalizer waits for rollout and settled orders

- **Decision**: Extend the existing `shophub.io/in-use-protection` finalizer's release. A Wallet
  being deleted is released only when **all** hold: no Shop references it (002, unchanged); the
  owning shop (label `shophub.io/shop`) is fully rolled out with no copy on the old address
  (same check as R10); and the shop answers `GET /internal/wallets/{address}/waiting` with
  `0`. If the owning Shop no longer exists or is being deleted for real, it is released at once.
  The watcher keeps reading a Wallet until it is gone, so payments to it are still delivered.
  Checked every minute while held.
- **Rationale**: ShopHub's reconciler already deletes old generations by name and tolerates a
  delete that waits (004 data-model, `oldest_wallet_generation`); only the operator's release
  condition changes.

## R10 — D-5: Ready only with no old copies

- **Decision**: `availableReplicas` treats the rollout as in progress unless
  `observedGeneration >= generation`, `updatedReplicas >= desired`, `replicas == updatedReplicas`
  and, where the field exists, `terminatingReplicas == 0`.
- **Rationale**: in T025 the Shop went Ready while an old pod still served; `replicas` counts old
  pods, so equality closes that window.

## R11 — D-4: a ValidatingAdmissionPolicy refuses a Shop delete ShopHub didn't ask for

- **Decision**: The shop-operator chart ships a `ValidatingAdmissionPolicy`
  (`admissionregistration.k8s.io/v1`, GA since Kubernetes 1.30; the local cluster runs 1.36.1
  and serves it) plus a `ValidatingAdmissionPolicyBinding` with `validationActions: [Deny]`:
  - match: `DELETE` of `shops.shophub.io` in the shops namespace (binding's
    `matchResources.namespaceSelector` on `kubernetes.io/metadata.name`);
  - validation: `has(oldObject.metadata.annotations) &&
    'shophub.io/delete-requested' in oldObject.metadata.annotations &&
    oldObject.metadata.annotations['shophub.io/delete-requested'] == 'true'`;
  - message: "Shops are deleted through ShopHub; this Shop has no shophub.io/delete-requested
    annotation."
  - exemption (`matchConditions`): requests from the Kubernetes controllers that delete on the
    cluster's behalf are not evaluated, so deleting the shops namespace or a teardown never hangs:
    `system:serviceaccount:kube-system:namespace-controller`, `system:serviceaccount:kube-system:generic-garbage-collector` and `system:kube-controller-manager` (the last one for clusters whose controller manager doesn't
    use per-controller credentials; docker-desktop runs with
    `--use-service-account-credentials=true`).
  ShopHub patches `shophub.io/delete-requested: "true"` onto the Shop right before deleting it
  (its Role already has `patch` on Shops). A `kubectl delete shop` without it is refused by the
  API server, so the Shop, its workloads and its data are never touched.
- **Rationale**: nothing is deleted, so nothing has to be restored: the shop keeps its data and
  keeps serving. No webhook server or certificates (the API server evaluates the CEL rule
  itself). Principle V stays as written: deleting a Shop (now only possible through ShopHub)
  still removes its derived resources via owner references.
- **Notes**: it guards against mistakes, not against cluster-admins (who can remove the binding);
  thanks to the exemption it does not block the namespace controller or the garbage collector,
  so deleting the shops namespace (or a whole teardown) completes; a person deleting a Shop
  directly is still refused. The policy and binding are platform state in the shop-operator chart, so
  uninstalling the chart removes them. CI clusters need ≥ 1.30: ShopHub's k3s tests pin
  `rancher/k3s:v1.36.1-k3s1`; the helm-charts and kube-state `install`/`e2e` jobs use
  `helm/kind-action@v1.12.0` with its default node image (≥ 1.30 but not pinned), so the chart
  PR pins the kind node image explicitly.
- **Alternatives**: an operator finalizer that orphans and re-adopts the data on an unrequested
  delete (works, but the Shop and its workloads still go down, Principle V's "deleting removes
  derived resources" gets an exception, and orphan/adopt is intricate code); holding the Shop in
  Terminating with a finalizer (ShopHub can't recreate a terminating object); a validating
  webhook (a server and certificates for one rule).

## R12 — D-6: ShopHub ignores a status older than the spec

- **Decision**: `KubeGateway.GetShopStatusAsync` treats the Shop as not ready (reason "Applying
  your change") unless `status.observedGeneration >= metadata.generation` and the Ready
  condition's `observedGeneration` (when present) is current.
- **Rationale**: the PATCH answer and the next list read happen before the operator reconciles.

## R13 — Egress: an explicit allowance for the operator (FR-020)

- **Decision**: The shop-operator chart adds an egress NetworkPolicy for the operator pod (new
  template `networkpolicy.yaml`; only the shophub chart has a fixed file list):
  DNS (53 UDP/TCP), TCP 443 and 6443 to anywhere (the provider's HTTPS endpoint and the API
  server), TCP 8081 to pods in `shops`. Shop pods keep ingress-only policies; the per-Shop policy
  adds TCP 8081 from the operator's namespace and pod labels.
- **Rationale**: today nothing restricts egress; writing the allowance down as policy means a
  later default-deny can't silently stop payment detection.

## R14 — Manual steps (recorded in Complexity Tracking)

1. **Provider account** (maintainer): a free Sepolia endpoint at a provider (Infura or Alchemy).
2. **RPC Secret** (maintainer, once per cluster):
   `kubectl create secret generic shop-operator-rpc -n shop-operator-system --from-literal=url=…`
   — never in git or chat. Without it the watcher idles and Wallets report
   `PaymentsWatching=False, reason RPCCredentialsMissing`.
3. **Remix deploy** (maintainer): deploy `MockUSDT.sol` on Sepolia from their MetaMask; put the
   address in `kube-state/clusters/local/shop-operator/values.yaml` (`payments.tokenAddress`).
