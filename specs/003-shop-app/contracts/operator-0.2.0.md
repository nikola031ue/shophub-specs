# Contract: shop-operator 0.2.0 changes (Phase 1)

Changes against [002 contracts/crds.md](../../002-shop-operator/contracts/crds.md) and the 002
resource builders. The CRDs (`shophub.io/v1alpha1`) are **unchanged**. When this ships, 002's
`contracts/crds.md` gets a "0.2.0" note that links here.

## Derived resources per Shop

| Resource | When | Name | Content | Owner |
|----------|------|------|---------|-------|
| Secret | every Shop | `<shop>-admin` | key `password`: 24 random URL-safe chars | Shop (controller) |
| Secret | `database: light` | `<shop>-redis-auth` | key `password`: 32 random URL-safe chars | Shop (controller) |
| OT `Redis` (changed) | `database: light` | `<shop>-redis` | see below | Shop (controller) |

Both Secrets carry `app.kubernetes.io/managed-by=shop-operator` plus the Shop's
`shophub.io/owner` / `shophub.io/shop` labels. They are created once with `Create`. They are never
updated with new values while they exist.

## Application container contract (what the app may rely on)

| Env | `standard` | `light` | Source |
|-----|-----------|---------|--------|
| `SHOP_NAME` | ✓ | ✓ | Shop `spec.shopName` (unchanged) |
| `WALLET_ADDRESS` | ✓ | ✓ | Wallet status (unchanged) |
| `DATABASE_URL` | ✓ | – | `secretKeyRef` `<shop>-db-app` / `uri` (unchanged) |
| `REDIS_URL` | – | ✓ | `redis://<shop>-redis.<ns>.svc:6379`, no credentials (unchanged) |
| `ADMIN_PASSWORD` | ✓ | ✓ | **new**: `secretKeyRef` `<shop>-admin` / `password` |
| `REDIS_PASSWORD` | – | ✓ | **new**: `secretKeyRef` `<shop>-redis-auth` / `password` |

- **Port** `80` (`http`); readiness = TCP connect to `http` (unchanged).
- **Memory**: **changed**: request `128Mi`, limit `256Mi` (0.1.0: request 32Mi, limit 128Mi).
  CPU request `10m` unchanged. See research R16.
- **New**: pod `securityContext.sysctls` `net.ipv4.ip_unprivileged_port_start=0`, so a non-root
  app can bind 80 (research R5).
- **New**: pod-template annotations `shophub.io/admin-secret-uid` and (light)
  `shophub.io/redis-secret-uid` hold the Secrets' UIDs. A recreated Secret therefore rolls the
  pods. UIDs only, never values or hashes.

## OT `Redis` spec additions (`light`)

```yaml
spec:
  kubernetesConfig:
    redisSecret: {name: <shop>-redis-auth, key: password}
  storage:
    keepAfterDelete: false
    volumeClaimTemplate:
      spec:
        accessModes: [ReadWriteOnce]
        resources: {requests: {storage: 1Gi}}
  redisConfig:
    dynamicConfig: ["appendonly yes"]
```

**PVC labels**: the OT CRD declares `volumeClaimTemplate.metadata` as an object without
properties, so the API server prunes labels set there (found in T002 envtest). This isn't needed
anyway: the Redis operator copies the Redis CR's labels (`managed-by`, `shophub.io/owner`,
`shophub.io/shop`) onto the PVC. That was verified on the cluster in T005.

## Reconcile rules added

1. Ensure `<shop>-admin` exists before applying the Deployment. On a cache miss, confirm with
   `APIReader`. If it is really absent, create it with a new random value. A regenerated admin
   Secret is safe (research R11).
2. `light` only: ensure the Redis password Secret exists before applying the Redis.
   - The current name is the one the live Redis references in
     `spec.kubernetesConfig.redisSecret.name`, or `<shop>-redis-auth` when there is no Redis yet.
   - **0.2.0**: if it is absent **and** `<shop>-redis` already exists, do not regenerate. Report
     `Ready=False` with reason `RedisCredentialsLost`; recovery is to recreate the Shop.
   - **0.2.1 (T026)**: create a new password under the next name instead:
     `<shop>-redis-auth` → `<shop>-redis-auth-2` → `-3` …; the old name is not reused.
     - Point the Redis at it. The OT redis-operator rolls the Redis pod with the new password and
       keeps the PVC and its AOF data (verified live with v0.27.0). No pod permissions are needed.
     - The Redis CR and the PVC are never deleted for this.
     - The Deployment's `REDIS_PASSWORD` `secretKeyRef` and `shophub.io/redis-secret-uid` follow
       the new Secret, so the app pods roll.
     - Light-shop readiness also waits for the Redis StatefulSet rollout (`observedGeneration`,
       `currentRevision == updateRevision`): `DatabaseProvisioning` while it restarts.
     - The info log names the missing and the new Secret, never a value.
3. **0.1.0 upgrade**: a live `<shop>-redis` without `spec.storage` is deleted once, with an info
   log `recreating Redis without storage (0.1.0 upgrade)`, and recreated on the next reconcile.
   The Shop shows `DatabaseProvisioning` meanwhile.
4. The Shop controller also `Owns` Secrets, so a deleted admin Secret is noticed and recreated
   without waiting for a resync.

## Status additions

| Kind | Reason | Ready | Meaning |
|------|--------|-------|---------|
| Shop | `RedisCredentialsLost` | False | **0.2.0 only**: light shop's Redis password Secret was deleted while its Redis exists; not regenerated. Retired in 0.2.1, which regenerates it (rule 2) and reports `DatabaseProvisioning` while the Redis restarts |

The operator emits no Kubernetes events (as in 0.1.0). No secret value ever appears in status,
events or logs.

## RBAC and chart

- **RBAC**: no new rules. Secrets already have full verbs, the Redis operator manages the PVC, and
  no events are emitted.
- **Image**: `ghcr.io/nikola031ue/shop-operator:0.2.0`.
- **Chart**: helm-charts `shop-operator` `0.3.0`, appVersion `0.2.0`. kube-state
  `clusters/local/shop-operator/helm.yaml` → `0.3.0`.

## Guarantees (added)

- Deleting a Shop also removes its admin and Redis Secrets and, through the Redis operator, the
  Redis PVC (data lost), within 2 minutes.
- Upgrading from 0.1.0 needs no hand change. Standard shops keep their database. Light shops get a
  fresh, authenticated, persistent Redis.
- An existing admin or Redis password is never replaced while its Secret exists.
