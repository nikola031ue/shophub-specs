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
- A shop that fails (timeout 5 s, non-2xx) keeps its cursor; condition
  `PaymentsWatching=False, reason ShopUnreachable`; others go on.
- Provider errors: back off (12 s → 2 min), condition `ProviderError`; no cursor moves.
- No RPC Secret: idle, `RPCCredentialsMissing`. No `tokenAddress`: idle, `TokenNotConfigured`.
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
patches the annotation, then deletes; cascade unchanged.

## Shop app wiring

Env `TOKEN_ADDRESS`, `CHAIN_ID`, `EXPLORER_TX_URL`, `ORDER_PAYMENT_MINUTES`, `INTERNAL_TOKEN`;
Service port `internal` 8081; NetworkPolicy ingress TCP 8081 from the operator's pods.

## Chart `shop-operator` 0.5.0

Values in data-model.md; `payments.rpcSecretName` referenced by `secretKeyRef` (optional);
new templates `admission-policy.yaml` (the guard above) and `networkpolicy.yaml` (egress for
the operator pod: DNS, TCP 443/6443 anywhere, TCP 8081 to `shops`, research R13). Operator RBAC
unchanged.
