# Contract: Shop internal API (operator → shop)

Served by every shop copy on port **8081** (`internal`), a separate Kestrel endpoint. The Ingress
never routes to it; the shop's NetworkPolicy admits it only from the operator's pods; every
request needs `Authorization: Bearer <INTERNAL_TOKEN>` (Secret `<shop>-internal`, key `token`),
compared in constant time. Wrong or missing token: 401. Not on port 8080; port 8080 answers 404
for `/internal/*`.

## POST /internal/chain

Delivers the confirmed transfers to one of this shop's addresses for a block range, and how far
the chain has been read for it. Sent on every watcher pass (empty `transfers` = heartbeat).

```json
{
  "address": "0x…",
  "fromBlock": 7000001,
  "seenThrough": {"block": 7000010, "time": "2026-11-27T10:00:00Z"},
  "transfers": [
    {"tx": "0x…", "logIndex": 3, "block": 7000004, "blockTime": "2026-11-27T09:58:48Z",
     "from": "0x…", "to": "0x…", "value": "11000000"}
  ]
}
```

- Answer **204** once every transfer is recorded (paid or unmatched) and `chain_progress` for
  `address` is at least `seenThrough`. Only then does the operator advance its cursor.
- Idempotent: a transfer already recorded (same `tx` + `logIndex`) is skipped; `seenThrough`
  never moves backwards.
- 400 when `address` is neither the shop's current `WALLET_ADDRESS` nor an address in its orders
  or `chain_progress`, or a transfer's `to` differs from `address`: the shop records nothing.
- Matching per transfer, in block order (research R6):
  - **paid**: an order of this shop with `payingAccount = from`, `walletAddress = to`, status
    `PendingPayment`, `totalBaseUnits = value`, `createdAt ≤ blockTime ≤ deadline`;
  - otherwise **unmatched**, with reason `late` (that order exists but expired or `blockTime` is
    past its deadline), `duplicate` (that order is already paid), `amount` (the account has a
    waiting order with another total), `account` (no order of this account), `unknown`.

## GET /internal/wallets/{address}/waiting

```json
{"address": "0x…", "waiting": 2}
```

Number of orders with this `walletAddress` still `PendingPayment`. The operator holds an old
Wallet's deletion until this is `0` (D-1, research R9).
