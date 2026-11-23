# Quickstart & Validation: Shop Application

Commands assume Git Bash. References: [operator contract](contracts/operator-0.2.0.md),
[HTTP API](contracts/http-api.md), [delivery](contracts/delivery.md),
[data model](data-model.md).

## Prerequisites

- The 002 platform is running from kube-state (`5 unchanged`), with the Discord token Secret in
  place and `platform-alerts` Ready.
- To develop the app: .NET 10 SDK (version from `shop/global.json`), Node LTS, and Docker Desktop
  (Testcontainers).
- Test resources are applied by hand **only** in the throwaway namespace `qs`, as in 002, and
  deleted at the end (plan: Complexity Tracking).

## Scenario 1 — Operator 0.2.0 tests (Phase 1)

```bash
cd shop-operator && go test ./...
```

Expected: all pass. The new envtest specs cover:

- admin Secret created, owned and referenced, and regenerated after deletion;
- for light shops, Redis Secret, `redisSecret`, storage and `appendonly`;
- `RedisCredentialsLost`;
- the 0.1.0 Redis recreate;
- the sysctl and the UID annotations.

## Scenario 2 — Automatic upgrade of 0.1.0 Shops (Phase 1)

1. Before the operator change, on the cluster still at operator 0.1.0: create a standard Shop
   `std` and a light Shop `lite` in `qs`, and wait for both to be Ready.
2. Merge the kube-state PR (`shop-operator` 0.3.0) and run `./bring-up.sh`. Expected: only
   `shop-operator` upgraded.
3. Expected within 5 min, with no hand change:
   - Both shops are Ready again and have `<shop>-admin` Secrets.
   - `lite` has `lite-redis-auth`. Its Redis was recreated once (operator log line) and has a
     Bound PVC.
   - `redis-cli -a` without a password gets `NOAUTH`; with the password it gets `PONG`.
   - `CONFIG GET appendonly` returns `yes`.
   - `std` database pod UID is unchanged.
4. Delete the `lite-redis-auth` Secret. Expected: `lite` shows `RedisCredentialsLost` and no new
   Secret appears. Recover by deleting and recreating `lite`.

## Scenario 3 — App tests (both stores, SC-003, SC-004)

```bash
cd shop && dotnet test && (cd src/Shop.Web && npm ci && npx ng test --watch=false)
```

Expected: the store contract suite passes for both PostgreSQL and Redis, including:

- 20 concurrent single-piece orders against 5 in stock → exactly 5 succeed;
- the lockout tests;
- the API tests on both stores.

## Scenario 4 — Release and rollout (US4, SC-008)

1. Tag `v0.1.0` in `shop`. Expected: the `release` job pushes `ghcr.io/nikola031ue/shop:0.1.0`
   within 15 min. After the package is made public, `docker pull` works anonymously.
2. Merge the kube-state PR that sets `shopImage` and run `./bring-up.sh`. Expected: only
   `shop-operator` upgraded; `std` and `lite` roll to the app and stay Ready.

## Scenario 5 — Admin and items (US1, SC-001, SC-006)

```bash
kubectl port-forward -n qs svc/std 8080:80
kubectl get secret -n qs std-admin -o jsonpath='{.data.password}' | base64 -d   # don't paste it anywhere
```

1. Open `http://localhost:8080/admin` and sign in. Expected: a prompt to change the initial
   password. Change it.
2. Create, edit and delete items as in US1 scenarios 2–4. Invalid input is refused per field.
3. Six wrong passwords in a row: the sixth attempt, even if correct, gets "locked" (429).
4. Repeat steps 1–2 on `lite`. Time from Ready to first visible item < 2 min (SC-001).
5. Restart the app pods (`kubectl rollout restart` is a hand change, so do it only in `qs`) and
   sign in again with the **changed** password. Expected: it works, because the changed password
   is not overwritten.

## Scenario 6 — Customer order (US2, SC-002)

On `std` and `lite`:

1. Search `shirt`, add 2 pieces to the cart, reload the page; the cart is kept.
2. Place the order. Expected: a reference is shown and the cart is empty; stock went down by 2.
   Time from opening the shop to the placed order < 1 min.
3. Fill a cart, have the admin change that item's price in another tab, then place the order.
   Expected: a "price changed" problem, no order created. After confirming the corrected cart,
   the order succeeds.

## Scenario 7 — Orders list (US3)

Sign in as admin and open Orders. Expected:

- the orders are listed newest first with reference, time, PendingPayment, total and lines;
- an order whose item was later renamed still shows the old name and price.

## Scenario 8 — Durability and replicas (SC-009, US4 scenario 6)

1. `std` with `availability: high` (3 replicas): sign in and click through items several times.
   The session keeps working on every replica.
2. Delete the `lite-redis` pod (a hand change in `qs`). After it returns, all items and orders
   are still there.
3. Upgrade the app (tag `v0.1.1` → kube-state tag change) or restart the pods. All data is kept on
   both shops.

## Scenario 9 — No secret leaks (SC-007, FR-024)

Scan for the admin and Redis passwords and the DB URI. Expected 0 matches in all of:

- `kubectl get shop -n qs -o yaml`;
- the app logs (`kubectl logs -n qs deploy/std`);
- the operator log;
- `git grep` over all repositories.

Clean up: `kubectl delete ns qs`.
