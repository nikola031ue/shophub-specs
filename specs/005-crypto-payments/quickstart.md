# Quickstart: Crypto Payments (005)

How to prove the feature end to end. Contracts: [shop-api](contracts/shop-api.md),
[shop-internal-api](contracts/shop-internal-api.md), [operator](contracts/operator.md),
[musdt](contracts/musdt.md); data: [data-model.md](data-model.md).

## Manual steps (maintainer; never automated, never in chat)

| # | Step | Done when |
|---|------|-----------|
| M1 | **Provider account**: create a free Sepolia endpoint at Infura or Alchemy | an HTTPS endpoint URL with key exists |
| M2 | **RPC Secret**, once per cluster: `kubectl create secret generic shop-operator-rpc -n shop-operator-system --from-literal=url="$SEPOLIA_RPC_URL"` | Secret exists (length check only) |
| M3 | **Remix deploy** of `contracts/MockUSDT.sol` on Sepolia from the maintainer's MetaMask (settings in `contracts/README.md`); verify on the explorer | address recorded in kube-state `payments.tokenAddress` via a PR |
| M4 | **Tester wallets**: two MetaMask accounts (A, B) with Sepolia ETH from a public faucet | each has ETH for gas |

## Scenario 1 — Automated (CI, local chain only; FR-019)

- shop: matching, uniqueness, expiry guard, idempotent receipt, internal auth, on PostgreSQL and
  Redis (Testcontainers).
- shop-operator: token on Anvil (mint cap, Transfer); watcher paging, confirmations, cursor and
  catch-up against Anvil with a fake shop endpoint; Wallet finalizer (D-1); Shop finalizer
  orphan/adopt (D-4); readiness (D-5) in envtest.
- shophub: stale status (D-6), annotation before delete (D-4).
- Expected: no CI job resolves a Sepolia host (checked by a CI step that greps the workflow and
  test settings for `sepolia`/provider hosts).

## Scenario 2 — Rollout

After the kube-state PR (operator chart 0.5.0 with `payments.*`, shop 0.3.0, ShopHub 0.3.0):
`./bring-up.sh`. Expected: Wallets report `PaymentsWatching=True` within 30 s; their
`scannedThroughBlock` follows Sepolia's head minus 6; operator logs show the provider host only.

## Scenario 3 — Pay (US1, SC-001, SC-002)

1. Account A: "Get test mUSDT" on a shop's order page (mint 1,000).
2. Checkout with A connected. Then try a second checkout with A: refused, naming the waiting
   order's reference (FR-005a).
3. On the order page "Pay with MetaMask". Expected: "payment sent, confirming", then **Paid**
   within 2 min of confirming in MetaMask; the admin list shows the transaction link; the shop's
   address received exactly the total. A new checkout with A now works.

## Scenario 4 — Wrong payments (US1 scenario 5, edge cases)

From A: send the total + 0.01; from B: send the exact total for A's order; pay an already paid
order again. Expected: order states unchanged; the admin "not matched" list shows reasons
`amount`, `account`, `duplicate`.

## Scenario 5 — Expiry (US2, SC-003)

Place an order (shop on `high`, 3 copies), don't pay. Expected: Expired within 1 min after its
30-minute deadline; each item's stock back once. Then pay it: recorded `late`, order stays
Expired, stock unchanged.

## Scenario 6 — Outage catch-up (FR-008, FR-011, SC-004)

Scale the operator to 0 (hand change, recorded), pay one order and leave another unpaid past its
deadline, wait 15 min, scale back to 1. Expected: the paid order is Paid within 5 min; the unpaid
one expires only after the watcher has passed its deadline; no order paid in time is Expired.

## Scenario 7 — Wallet change with a waiting order (US3, D-1, D-5, D-6, SC-005, SC-006)

Place an order on the old address; change the shop's wallet in the panel. Expected:
- the panel shows "being prepared" within 5 s (D-6);
- the Shop is not Ready while any old copy runs (D-5); new orders after Ready carry the new
  address;
- Wallet `<id>-w1` stays (`WaitingForOrders`) while that order waits (D-1); paying it to the old
  address marks it Paid; `-w1` is gone within 5 min after.

## Scenario 8 — Delete by hand keeps the data (D-4, SC-007)

Note a shop's items, orders and the admin Secret's UID; `kubectl delete shop <id> -n shops` (hand
change, recorded). Expected: event `DataKept`; ShopHub recreates the Shop; same items, orders,
admin password (Secret UID unchanged), same database volume.

## Scenario 9 — Delete through ShopHub (004 FR-013)

Delete a shop in the panel. Expected: everything gone within 2 min (Shop, Wallets, channel,
database, `<id>-internal`), as in 004.

## Scenario 10 — Secrets (FR-018, SC-008)

As 004 Scenario 9, adding the RPC URL and every `<shop>-internal` token. Expected 0 matches in
every repository's history, logs (operator, shop, ShopHub), resource YAML and API responses.
