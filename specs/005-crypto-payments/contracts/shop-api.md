# Contract: Shop app API changes (repo `shop`, 0.3.0)

Public and admin routes of spec 003, changed or added for payments. Amounts are strings of
mUSDT base units (6 decimals) plus a display string, so no client does floating point.

## Public

| Method | Path | Body | Answers |
|--------|------|------|---------|
| GET | `/api/shop` | – | 200 adds `payments: {tokenAddress, chainId, explorerTxUrl, mintPerCall: "1000000000"}`; `tokenAddress` null when not configured |
| POST | `/api/orders` | `{lines, payingAccount}` | 201 as 003 plus the fields below · 400 `payingAccount` missing or not an address · 409 `reason: waitingOrder, reference` when this account already has an order waiting in this shop (FR-005a) · 409 `reason: paymentsUnavailable` when the token isn't configured · other 003 answers unchanged |
| GET | `/api/orders/{reference}` | – | 200 `Order` (below) · 404. The reference is the buyer's handle (003); no other lookup |

`Order` (public):

```json
{
  "reference": "34JV98Z2TH",
  "status": "PendingPayment | Paid | Expired",
  "lines": [{"name": "…", "unitPriceCents": 1100, "quantity": 1}],
  "totalCents": 1100,
  "totalBaseUnits": "11000000",
  "walletAddress": "0x…",
  "payingAccount": "0x…",
  "createdAt": "…", "deadline": "…",
  "payment": {"tx": "0x…", "block": 123, "paidAt": "…", "explorerUrl": "https://sepolia.etherscan.io/tx/0x…"}
}
```

`payment` only when paid. Status values are spec 003's (`PendingPayment`) plus `Paid` and `Expired`,
kept in that form so existing clients don't change. The page polls this every 5 s while
`PendingPayment`.

## Admin (session as 003)

| Method | Path | Answers |
|--------|------|---------|
| GET | `/api/admin/orders` | 200, each order as above (status, deadline, payment with explorer link) |
| GET | `/api/admin/payments?matched=false` | 200 `{payments: [{tx, logIndex, block, blockTime, from, to, valueBaseUnits, reason, explorerUrl}]}`, newest first (FR-007, FR-016) |

## Angular (storefront)

- **Checkout**: "Connect MetaMask" (EIP-1193 `eth_requestAccounts`), then "Place order" sends
  `payingAccount`. 409 `waitingOrder` links to that order.
- **Order page** `/orders/{reference}`: total in mUSDT, address, paying account, time left,
  status. "Pay with MetaMask":
  1. `wallet_switchEthereumChain` to `chainId` (FR-002, scenario 2), nothing sent until switched;
  2. the connected account must equal `payingAccount`, else say so and send nothing;
  3. read `balanceOf(account)` and the ETH balance; missing mUSDT or ETH → say which (scenario 3);
  4. `eth_sendTransaction` of `transfer(walletAddress, totalBaseUnits)` on the token.
  After MetaMask returns a hash the page shows "payment sent, confirming" (kept in
  `localStorage` per reference) until the order turns `paid` (FR-006).
- **Get test mUSDT**: sends `mint(1000000000)` (1,000 mUSDT) from the connected account (FR-017).
- No wallet library: EIP-1193 calls and ABI encoding of `transfer`/`mint`/`balanceOf` by hand
  (three fixed selectors).
- Admin orders page: status column, payment link; a "Payments not matched" list.
