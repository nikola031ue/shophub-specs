# Data Model: Shop Operator

API group/version `shophub.io/v1alpha1`; all kinds namespaced. Field-level contract with
printer columns and reason codes: [contracts/crds.md](contracts/crds.md).

## Shop

| Field | Type | Rules |
|-------|------|-------|
| `metadata.name` | string | DNS-1123 label, ≤ 40 chars (children append `-db`, `-redis`) |
| `metadata.labels["shophub.io/owner"]`, `["shophub.io/shop"]` | string | Copied to every child (FR-019) |
| `spec.shopName` | string | Required, 1–100 chars; immutable (CEL `self == oldSelf`) |
| `spec.availability` | enum | Required: `standard` \| `high` → 2 \| 3 replicas |
| `spec.database` | enum | Required: `standard` (PostgreSQL) \| `light` (Redis); immutable |
| `spec.walletRef.name` | string | Required; Wallet in same namespace |
| `spec.discordChannelRef.name` | string | Required; DiscordChannel in same namespace |
| `status.conditions[Ready]` | Condition | `status`, `reason`, `message`, `lastTransitionTime` |
| `status.observedGeneration` | int64 | Last processed `metadata.generation` |
| `status.replicas` / `status.readyReplicas` | int32 | Desired / available copies |
| `status.url` | string | `http://<name>.<ns>.svc.cluster.local` |
| `status.walletAddress` | string | Address in use (from Wallet status) |
| `status.webhookSecretName` | string | From the referenced DiscordChannel's status |

**Owns** (owner reference, `controller: true`, garbage-collected on delete — FR-006):

- Deployment `<name>`
- Service `<name>`
- CNPG `Cluster` `<name>-db` (1 instance, 1 Gi), when `database=standard`
- OT `Redis` `<name>-redis` (standalone), when `database=light`

**State transitions** (Ready reason):

```text
            refs missing/not ready ─► WalletNotReady | DiscordChannelNotReady
submitted ─► database kind missing  ─► DatabaseOperatorMissing
            database provisioning  ─► DatabaseProvisioning
            pods starting          ─► Progressing
            all ready              ─► Ready=True (reason Ready)
any child error ─► ReconcileError (message = cause); retried with backoff
```

## Wallet

| Field | Type | Rules |
|-------|------|-------|
| `spec.address` | string | Optional; `^0x[0-9a-fA-F]{40}$`; immutable once set (CEL); EIP-55 checked by controller if mixed case |
| `status.conditions[Ready]` | Condition | Reasons below |
| `status.address` | string | Registered or generated address (checksummed) |
| `status.mode` | enum | `registered` \| `generated` |
| `status.keySecretName` | string | `<name>-wallet-key` (generated mode only) |
| `status.usedBy` | []string | Names of Shops referencing it |

**Owns**: Secret `<name>-wallet-key` (key `privateKey`), generated mode only.
**Finalizer**: `shophub.io/in-use-protection` blocks deletion while `usedBy` is non-empty.

```text
address given  ─► checksum ok ─► Ready (registered)      │ bad checksum ─► InvalidChecksum
address absent ─► key Secret exists? yes ─► derive address ─► Ready (generated)
                                     no  ─► generate key, create Secret ─► Ready (generated)
delete requested ─► usedBy non-empty ─► InUse (waits) ─► empty ─► finalizer removed ─► GC Secret
```

## DiscordChannel

| Field | Type | Rules |
|-------|------|-------|
| `spec.channelName` | string | Required; `^[a-z0-9-]{1,100}$` |
| `status.conditions[Ready]` | Condition | Reasons below |
| `status.channelID` | string | Discord channel snowflake |
| `status.webhookSecretName` | string | `<name>-discord-webhook` |
| `status.usedBy` | []string | Shops referencing it |

**Owns**: Secret `<name>-discord-webhook` (key `url`).
**Finalizer**: `shophub.io/discord-cleanup` removes the channel from Discord after `usedBy`
is empty. A Discord `404` on delete counts as success.

```text
no token Secret ─► DiscordCredentialsMissing
channelID empty ─► find channel by topic "shophub-uid:<uid>" ─► found: adopt │ not found: create
webhook Secret missing ─► create webhook, write Secret ─► Ready
Discord 429/5xx/network ─► DiscordUnavailable (retry after retry_after / backoff)
Discord 401/403 ─► DiscordForbidden
```

## Relationships

```text
Wallet 1 ◄── * Shop * ──► 1 DiscordChannel
                │
                ├─ owns Deployment, Service
                └─ owns CNPG Cluster  xor  OT Redis
```

## Platform configuration (operator flags ← chart values)

| Flag | Chart value | Default |
|------|-------------|---------|
| `--shop-image` | `shopImage.repository` | `nginx` |
| `--shop-image-tag` | `shopImage.tag` | `1.27.3` (SemVer, never `latest`) |
| `--discord-guild-id` | `discord.guildID` | `""` (DiscordChannels report `DiscordCredentialsMissing`) |
| `--discord-token-secret` | `discord.tokenSecret` | `shop-operator-discord` (in release namespace, key `token`) |
