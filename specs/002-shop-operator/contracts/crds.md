# Contract: Custom Resources (`shophub.io/v1alpha1`)

The interface the future ShopHub backend codes against. Fields and rules:
[data-model.md](../data-model.md).

## Examples

```yaml
apiVersion: shophub.io/v1alpha1
kind: Wallet
metadata: { name: acme-wallet, namespace: user-42 }
spec: {}                      # omit address → generated; or address: "0xAbC…" → registered
---
apiVersion: shophub.io/v1alpha1
kind: DiscordChannel
metadata: { name: acme-alerts, namespace: user-42 }
spec: { channelName: acme-alerts }
---
apiVersion: shophub.io/v1alpha1
kind: Shop
metadata:
  name: acme
  namespace: user-42
  labels: { shophub.io/owner: user-42, shophub.io/shop: acme }
spec:
  shopName: "Acme Clothing"
  availability: standard      # standard | high
  database: standard          # standard (PostgreSQL) | light (Redis)
  walletRef: { name: acme-wallet }
  discordChannelRef: { name: acme-alerts }
```

## Admission rejections (SC-004; CRD schema + CEL, no webhook)

| Input | Message contains |
|-------|------------------|
| `availability: medium` | `Unsupported value: "medium"`, `"standard", "high"` |
| missing `walletRef` / `discordChannelRef` / `shopName` | `Required value` and the field path |
| change `database` or `shopName` | `database is immutable` / `shopName is immutable` |
| Wallet `address: 0x123` | `should match '^0x[0-9a-fA-F]{40}$'` |
| change a set Wallet `address` | `address is immutable` |
| `channelName: Acme Alerts` | `should match '^[a-z0-9-]{1,100}$'` |

## Status conditions — `Ready` reason codes

| Kind | Reason | Ready | Meaning |
|------|--------|-------|---------|
| all | `Ready` | True | Fully reconciled |
| all | `ReconcileError` | False | Unexpected error; message has the cause |
| Shop | `WalletNotReady` / `DiscordChannelNotReady` | False | Ref missing or not Ready (message names it) |
| Shop | `DatabaseOperatorMissing` | False | CNPG or Redis kind not served by the cluster |
| Shop | `DatabaseProvisioning` | False | Database CR exists, not ready yet |
| Shop | `Progressing` | False | Pods not all available |
| Wallet | `InvalidChecksum` | False | Mixed-case address fails EIP-55 |
| Wallet | `KeyLost` | False | Generated Wallet's key Secret was deleted; old address kept, no new key generated |
| Wallet, DiscordChannel | `InUse` | False | Deletion waits for referencing Shops (`status.usedBy`) |
| DiscordChannel | `DiscordCredentialsMissing` | False | Token Secret or guild ID missing (names the Secret) |
| DiscordChannel | `DiscordUnavailable` | False | 429 / 5xx / network; retried |
| DiscordChannel | `DiscordForbidden` | False | 401 / 403 from Discord |

No secret value (private key, webhook URL, token) may appear in status, events or logs (SC-007).

## Printer columns

| Kind | Columns |
|------|---------|
| Shop | `AVAILABILITY`, `DATABASE`, `READY`, `REASON`, `URL`, `AGE` |
| Wallet | `ADDRESS`, `MODE`, `READY`, `AGE` |
| DiscordChannel | `CHANNEL`, `READY`, `REASON`, `AGE` |

## Guarantees

- Deleting a Shop removes its Deployment, Service and database within 2 minutes (SC-003);
  the database data is lost.
- Manual edits to owned children are reverted on the next reconcile (≤ 30 s while watched).
- Operator restarts never duplicate Discord channels (topic `shophub-uid:<uid>` lookup) and
  never regenerate an existing wallet key.
