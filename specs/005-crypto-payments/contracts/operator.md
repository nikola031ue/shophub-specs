# Contract: shop-operator 0.4.0 behaviour

## Chain watcher (research R1–R3)

- One manager Runnable, leader election only (1 active replica).
- Every `payments.pollSeconds` (12): `eth_blockNumber`; `confirmed = head − confirmations`.
- Wallets in `shops` with `status.payments.scannedThroughBlock` unset start at `confirmed`.
- `from = min(cursor) + 1`; for each range of ≤ `logsBlockRange` blocks up to `confirmed`:
  `eth_getLogs{address: tokenAddress, topics: [Transfer, null, [wallet topics]], fromBlock,
  toBlock}`; group logs by `to`; for every Wallet whose cursor is below the range end, post
  `/internal/chain` to its shop (label `shophub.io/shop`) with that Wallet's transfers in
  `(cursor, rangeEnd]` and `seenThrough = rangeEnd`; on 204 patch the Wallet's cursor.
- A shop that fails (timeout 5 s, non-2xx) keeps its cursor and gets `status.payments.lastError =
  ShopUnreachable`; others go on.
- Provider errors: back off (12 s → 2 min), `lastError = ProviderError`; no cursor moves.
- No RPC Secret: `lastError = RPCCredentialsMissing`. No `tokenAddress`: the watcher idles.
- The watcher writes only `status.payments`. The **Wallet controller** derives `PaymentsWatching`
  from it every minute: `TokenNotConfigured`, `Starting`, the watcher's `lastError`, **`Stalled`**
  (chain read further behind than `--payments-stall-after`, 5 m: a stopped watcher writes nothing,
  so the controller notices), else `Watching`.
- The RPC URL is never logged (only its host), never in status or events.

## Wallet deletion (D-1, research R9)

Finalizer `shophub.io/in-use-protection` is removed only when no Shop references the Wallet
(002), **and** either the owning Shop (label `shophub.io/shop`) is gone or being deleted with
`shophub.io/delete-requested`, **or** it is fully rolled out (R10) and
`GET /internal/wallets/{address}/waiting` answers `0`. While held: `Ready=False`,
reason `WaitingForOrders`, requeue every 60 s. The watcher keeps reading the Wallet until it is
gone.

## Shop readiness (D-5, research R10)

Rollout in progress unless `observedGeneration ≥ generation`, `updatedReplicas ≥ desired`,
`replicas == updatedReplicas` and no terminating replicas.

## Shop deletion guard (D-4, research R11)

Shipped by the chart (not the operator binary), `templates/admission-policy.yaml`:

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata: {name: shophub-shop-delete-guard}
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: ["shophub.io"]
        apiVersions: ["v1alpha1"]
        operations: ["DELETE"]
        resources: ["shops"]
  # Never block Kubernetes' own cleanup: namespace deletion and garbage collection.
  matchConditions:
    - name: not-a-cluster-controller
      expression: >-
        !(request.userInfo.username in [
          'system:serviceaccount:kube-system:namespace-controller',
          'system:serviceaccount:kube-system:generic-garbage-collector',
          'system:kube-controller-manager'])
  validations:
    - expression: >-
        has(oldObject.metadata.annotations) &&
        'shophub.io/delete-requested' in oldObject.metadata.annotations &&
        oldObject.metadata.annotations['shophub.io/delete-requested'] == 'true'
      message: Shops are deleted through ShopHub; this Shop has no shophub.io/delete-requested annotation.
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata: {name: shophub-shop-delete-guard}
spec:
  policyName: shophub-shop-delete-guard
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels: {kubernetes.io/metadata.name: <shopsNamespace>}
```

Rendered when `shopDeleteGuard.enabled` (default `true`). ShopHub's delete (004 FR-013) first
patches the annotation, then deletes; cascade unchanged. The `matchConditions` exemption lets the
namespace controller and the garbage collector delete Shops, so deleting the shops namespace or a
teardown completes; the chart's `install` job proves it (T020).

## Shop app wiring

Env `TOKEN_ADDRESS`, `CHAIN_ID`, `EXPLORER_TX_URL`, `ORDER_PAYMENT_MINUTES`, `INTERNAL_TOKEN`;
Service port `internal` 8081; NetworkPolicy ingress TCP 8081 from the operator's pods.

## Chart `shop-operator` 0.5.0

Values in data-model.md; `payments.rpcSecretName` referenced by `secretKeyRef` (optional);
new templates `admission-policy.yaml` (the guard above) and `networkpolicy.yaml` (egress for
the operator pod: DNS, TCP 443/6443 anywhere, TCP 8081 to `shops`, research R13). Operator RBAC
unchanged.
