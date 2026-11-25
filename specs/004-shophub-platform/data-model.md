# Data Model: ShopHub Platform (004)

## ShopHub database (PostgreSQL, CNPG `shophub-db`)

### `owners`

| Column | Type | Rules |
|--------|------|-------|
| `id` | uuid (v7) | PK. Stable identifier, used as the `shophub.io/owner` label value (FR-019) |
| `email` | text | Trimmed, stored as entered |
| `email_normalized` | text | `lower(trim(email))`, UNIQUE (FR-001) |
| `password_hash` | text | `PasswordHasher` (PBKDF2), never returned or logged (FR-004) |
| `created_at` | timestamptz | |

Validation: email has one `@` and is at most 254 characters; password is 12–200 characters.

### `owner_sessions`

| Column | Type | Rules |
|--------|------|-------|
| `id` | uuid | PK; carried in the cookie as a claim |
| `owner_id` | uuid | FK `owners`, cascade delete |
| `created_at`, `expires_at` | timestamptz | `expires_at = created_at + 8 h` (FR-002) |

Sign-out deletes the row. A password change deletes every other row of the owner (FR-002). Every
request checks the row exists and hasn't expired, so a copied cookie stops working after either.

### `signin_lockouts`

| Column | Type | Rules |
|--------|------|-------|
| `account` | text | `owners.email_normalized` form of the email tried; rows exist for unknown emails too |
| `address` | text | Real client address (R5/R7), or `*` for the account-wide row |
| `failed_count` | int | Failures in the current window |
| `failed_window_start` | timestamptz | Start of the 15 min window (first failure) |
| `locked_until` | timestamptz | Set when a limit is reached |

PRIMARY KEY `(account, address)`. Same fixed-window semantics as shop 0.2.0 (FR-003):

- A failure increments `(account, address)` (limit **5**) and `(account, '*')` (limit **50**),
  each within a 15 min window, unless either is locked; then nothing changes.
- Reaching a limit sets `locked_until = now + 15 min` and resets that counter.
- A sign-in is refused (429, `Retry-After`) while either row is locked, even with the right password.
- A successful sign-in clears only its `(account, address)` row.
- Rows whose window and lock are over are pruned on each failure.

### `shops` (Shop record, FR-015)

| Column | Type | Rules |
|--------|------|-------|
| `id` | text | PK. Generated identifier `s[a-z2-7]{9}` (R9) |
| `owner_id` | uuid | FK `owners` |
| `display_name` | text | 1–100 characters after trimming, counted as code points; no control characters (narrowing of FR-006: the name is shown in the panel and copied into the Shop) |
| `display_name_key` | text | `lower(trim(display_name))`; UNIQUE `(owner_id, display_name_key)` |
| `availability` | text | `standard` \| `high` |
| `database` | text | `standard` \| `light`; immutable |
| `wallet_address` | text | `^0x[0-9a-fA-F]{40}$`; mixed case must pass EIP-55 (FR-006, FR-009); stored in EIP-55 form |
| `wallet_generation` | int | Starts at 1; +1 on every address change. Current Wallet name is `<id>-w<generation>` |
| `oldest_wallet_generation` | int | Starts at 1; `1 ≤ oldest ≤ wallet_generation`. Lowest generation that may still exist in the cluster; generations below it are known deleted (T021, migration 003). Only the reconciler raises it, never lowers it |
| `state` | text | `Active` \| `Deleting` |
| `created_at`, `updated_at` | timestamptz | `updated_at` drives "needs attention" after 10 min not ready (R10) |

Rules:

- At most 5 rows per `owner_id`, in any state (FR-011, setting `MaxShopsPerOwner`). Checked
  under a row lock on the owner.
- Create is repeat-safe (SC-010): under the same lock, a request matching an existing row's
  trimmed `display_name`, `availability`, `database` and `wallet_address`, with the row `Active`,
  returns that row and inserts nothing. The same `display_name_key` with anything different is a
  duplicate name (409). The repeat check comes before the limit check.
- `display_name` and `database` never change after insert (FR-012).
- A change (FR-012) runs in one transaction under a row lock on the shop (`FOR UPDATE`). A
  different `wallet_address` (compared in EIP-55 form, so letter case alone is no change) adds 1
  to `wallet_generation`; the same address doesn't. Going back to an earlier address is a new
  generation too, since a Wallet's address never changes. `updated_at` moves only when a value
  changed; repeating the current values writes nothing. A `Deleting` row can't be changed.
- Delete (FR-013) sets `state = Deleting` and `updated_at` under the same row lock, after checking
  ownership and then that the typed name, trimmed, equals `display_name` exactly. The reconciler
  removes the row (`DELETE … WHERE state = 'Deleting'`) once none of the shop's objects exists.
  Until then the row still counts towards the limit and keeps its `display_name_key`.
- Every query for an owner-facing API filters on `owner_id = @current` (FR-005). A missing row and
  another owner's row give the same 404.

State transitions:

```text
(create) ──► Active ──(delete)──► Deleting ──(all resources gone)──► (row removed)
              │  ▲
              └──┘ change availability / wallet address
```

### `data_protection_keys`

Same as `shop`: `id`, `xml`, `created_at`.

## Cluster resources written by ShopHub (namespace `shops`)

All carry these labels:

- `shophub.io/owner: <owners.id>`
- `shophub.io/shop: <shops.id>`
- `app.kubernetes.io/managed-by: shophub`

They are applied with field manager `shophub`.

| Kind | Name | Spec from the record |
|------|------|----------------------|
| `DiscordChannel` | `<id>` | `channelName: <id>` |
| `Wallet` | `<id>-w<generation>` | `address: <wallet_address>` (registered only, FR-009) |
| `Shop` | `<id>` | `shopName: <display_name>`, `availability`, `database`, `walletRef.name: <id>-w<generation>`, `discordChannelRef.name: <id>` |

Read only (never written by ShopHub):

- Shop `.status` (`conditions[Ready]`, `url`, `readyReplicas`);
- Secret `<id>-admin` key `password`, only for FR-016. RBAC allows `get` on any Secret name in
  `shops`; reading only `<id>-admin` is enforced by the gateway (contracts/delivery.md).

Deletes name each object exactly, from the shop record: Shop `<id>`, DiscordChannel `<id>`, and
Wallets `<id>-w1` … `<id>-w<wallet_generation>`. Never a label selector or a collection delete.

After a wallet change (FR-009), the reconciler's apply creates Wallet `<id>-w<wallet_generation>`
and points the Shop at it. Only after that does it delete Wallets `<id>-w<oldest_wallet_generation>`
… `<id>-w<wallet_generation - 1>` by name, then set `oldest_wallet_generation = wallet_generation`.
A failed delete leaves the column as it was, so the next pass retries; a crash between the delete
and the update only repeats deletes of names already gone. The operator keeps a Wallet while a
Shop still uses it (002 `InUse`).

For a `Deleting` record each pass deletes Shop `<id>` first (the operator keeps a Wallet or
DiscordChannel while a Shop uses it), then DiscordChannel `<id>` and Wallets `<id>-w1` …
`<id>-w<wallet_generation>`, then reads each of those names back; the row goes when none
exists. While finalizers hold any of them, passes run every 5 s instead of 30 s (SC-003).

## Derived resources added by shop-operator 0.3.0 (owned by the Shop)

| Kind | Name | Content |
|------|------|---------|
| `Ingress` | `<shop>` | class from `--ingress-class`; host `<shop>.<--shop-domain>`; path `/` → Service `<shop>:80` |
| `NetworkPolicy` | `<shop>` | Pods `app.kubernetes.io/instance=<shop>`: ingress only from namespace `traefik` and from the ShopHub app pods in namespace `shophub` (namespace by `kubernetes.io/metadata.name` plus pod labels from `--shophub-pod-selector`, in one peer), TCP 80 |

Shop `status.url` becomes `https://<shop>.<shop-domain>`. The Deployment gets env
`TRUSTED_PROXY_HOST` (from `--trusted-proxy-host`, research R5). The Shop CRD rejects a
`metadata.name` containing a dot (FR-021).

## shop app 0.2.0 storage additions

| Store | Layout |
|-------|--------|
| PostgreSQL | Table `admin_lockouts (account text, address text, failed_count int, failed_window_start timestamptz, locked_until timestamptz, PRIMARY KEY (account, address))`. `account` is the signing-in account (`admin`, the shop's single admin account) and `address` the real client address; the row `('*', '*')` is the shop-wide ceiling. Migration `003_lockout_per_account_address.sql` creates it; the admin row's `failed_count`, `failed_window_start` and `locked_until` (001) stop being used |
| Redis | Hashes `shop:admin:lock:<account>:<address>` and `shop:admin:lock:*` with the same three fields; `failed_signin.lua` updates both scopes atomically. Keys expire 15 min after their last change |

Same fixed-window semantics as 003, applied per scope:

- A failure increments both the (account, address) scope (limit **5**) and `*` (limit **50**),
  each within a 15 min window.
- Reaching a limit sets that scope's `locked_until = now + 15 min`.
- A sign-in is refused (429, `Retry-After`) while its (account, address) scope or `*` is locked,
  even with the right password.
- A successful sign-in resets only its (account, address) scope (FR-023).
- The order rate limit is not stored here and stays keyed by client address only (FR-024).

## Panel state mapping (FR-008, R10)

| Record state | Shop Ready | Age since `updated_at` | Panel state |
|--------------|-----------|------------------------|-------------|
| `Deleting` | any | any | being deleted |
| `Active` | True | any | ready |
| `Active` | False / missing | < 10 min | being prepared |
| `Active` | False / missing | ≥ 10 min | needs attention |

The reason text comes from the Shop's `Ready` reason (contracts/panel-api.md).
