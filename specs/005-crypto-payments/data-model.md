# Data Model: Crypto Payments (005)

Changes only; the rest is as in specs 002–004.

## Shop app (repo `shop`): orders and payments

Both stores (PostgreSQL `standard`, Redis `light`) hold the same logical data.

### Order (extends 003)

| Field | Type | Rule |
|-------|------|------|
| `paying_account` | text, EIP-55 | Set at checkout from the connected MetaMask account (FR-002a); required for new orders |
| `total_base_units` | integer (numeric(30) / string in Redis) | `total_cents × 10^4` (mUSDT has 6 decimals, R7); fixed at placement |
| `deadline` | timestamptz | `created_at + ORDER_PAYMENT_MINUTES` (default 30, FR-009) |
| `status` | text | `PendingPayment` → `Paid` \| `Expired` (no other moves) |
| `paid_tx`, `paid_log_index`, `paid_block`, `paid_at` | text, int, bigint, timestamptz | Set when Paid; `paid_at` is the transfer's block time |

Existing `wallet_address` (003) is the address the order is paid to (FR-003).

**Uniqueness (FR-005a)**: at most one `PendingPayment` order per `paying_account` in a shop.
- PostgreSQL: `CREATE UNIQUE INDEX … ON orders (paying_account) WHERE status = 'PendingPayment'`.
- Redis: key `shop:waiting:<account>` → order reference, set with NX in `place_order.lua`, deleted
  in the paid and expire scripts.

Checkout refused for that reason answers 409 `reason: waitingOrder` with the waiting reference.

**State transitions** (each one atomic, conditional on `status = 'PendingPayment'`):
- **Paid**: matching transfer (R6), its block time ≤ `deadline`. Stock unchanged.
- **Expired**: `deadline < now` **and** `deadline < seen_through_time` for the order's
  `wallet_address` (R5). Returns each line's quantity to its item if the item still exists.

### Payment (new) — every delivered transfer to one of this shop's addresses

| Field | Type | Rule |
|-------|------|------|
| `tx_hash`, `log_index` | text, int | Primary key: receipt is idempotent |
| `block`, `block_time` | bigint, timestamptz | From the operator's delivery |
| `from_account`, `to_address` | text, EIP-55 | |
| `value_base_units` | numeric(30) | |
| `order_reference` | text, nullable | The order it paid, or null |
| `unmatched_reason` | text, nullable | `amount` \| `account` \| `late` \| `duplicate` \| `unknown` when not matched (FR-007, FR-010) |
| `received_at` | timestamptz | When the shop recorded it |

Redis: hash `shop:payment:<tx>:<logIndex>`, sorted set `shop:payments` by block time.

### Chain progress (new) — per receiving address

| Field | Type | Rule |
|-------|------|------|
| `address` | text, EIP-55, PK | A Wallet address of this shop (current or old) |
| `seen_through_block`, `seen_through_time` | bigint, timestamptz | Last value delivered; only moves forward |

### Schema scripts

- PostgreSQL `004_payments.sql`: the order columns (existing orders get `paying_account` NULL,
  `deadline = created_at + 30 min`, `total_base_units` from their total), the partial unique
  index, `payments`, `chain_progress`.
- Redis: new scripts `record_transfer.lua` (match-or-record, idempotent), `expire_order.lua`;
  `place_order.lua` gains the waiting key and the new fields. Old orders without
  `paying_account` can never match; they expire normally.

## Kubernetes resources (repo `shop-operator`)

### Wallet status (additions; CRD `shophub.io/v1alpha1`, status only)

| Field | Meaning |
|-------|---------|
| `status.payments.scannedThroughBlock` | Last confirmed block whose transfers the owning shop accepted (R3) |
| `status.payments.scannedThroughTime` | That block's time |
| condition `PaymentsWatching` | `True`; or `False` with reason `RPCCredentialsMissing`, `TokenNotConfigured`, `ShopUnreachable`, `ProviderError` |
| condition `Ready`, reason `WaitingForOrders` | Deletion held by D-1 (R9): message names the open-order count or the rollout |

Spec is unchanged (no CRD schema change beyond status fields; chart bumps the CRD).

### Shop

- Finalizer `shophub.io/data-protection` on every Shop (R11).
- Annotation `shophub.io/delete-requested: "true"`, written only by ShopHub immediately before
  its delete. Without it, a deletion orphans the data objects and the next create adopts them.
- Readiness (R10): rollout in progress unless `replicas == updatedReplicas` (and no terminating
  replicas) besides the existing checks.
- New env for the shop app: `TOKEN_ADDRESS`, `CHAIN_ID` (11155111), `EXPLORER_TX_URL`
  (`https://sepolia.etherscan.io/tx/`), `ORDER_PAYMENT_MINUTES`, `INTERNAL_TOKEN`
  (`secretKeyRef <shop>-internal/token`).
- Service gains port `internal` 8081 → container 8081. The Ingress keeps pointing at `http`.
- NetworkPolicy gains: TCP 8081 from pods of the operator (namespace `shop-operator-system`,
  operator pod labels).

### Derived Secret (new)

| Secret | Owner | Content |
|--------|-------|---------|
| `<shop>-internal` | Shop (orphaned and re-adopted under R11) | `token`: 32 random bytes, base64; generated once, never regenerated while it exists |

### Platform settings and Secrets (chart `shop-operator`, kube-state values)

| Value | Kind | Default |
|-------|------|---------|
| `payments.tokenAddress` | public setting (kube-state) | empty → watcher idle, `TokenNotConfigured` |
| `payments.chainId` | public | `11155111` |
| `payments.confirmations` | public | `6` |
| `payments.pollSeconds` | public | `12` |
| `payments.logsBlockRange` | public | `10` |
| `payments.orderPaymentMinutes` | public | `30` |
| `payments.explorerTxUrl` | public | `https://sepolia.etherscan.io/tx/` |
| `payments.rpcSecretName` | Secret reference | `shop-operator-rpc` (key `url`), **made by hand**, optional like the Discord token |

## ShopHub (repo `shophub`)

- No schema change.
- `KubeGateway.DeleteShopAsync`: patch the annotation, then delete (R11).
- `KubeGateway.GetShopStatusAsync`: not ready when the status is older than the spec (R12).
