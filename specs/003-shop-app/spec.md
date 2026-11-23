# Feature Specification: Shop Application (Requirements 2.1, 2.2, 2.3)

**Feature Branch**: `003-shop-app`

**Created**: 2026-11-23

**Status**: Draft

**Input**: User description: "Shop application (requirements 2.1, 2.2, 2.3 from ShopHub.md): one
deployable web app, Angular frontend + .NET backend, one container image, deployed per shop by the
existing shop-operator 0.1.0. Admin manages items (name, stock, price) and lists orders; customers
search items, use a cart and place an order that stays in status PendingPayment. The app MUST run
on both database modes the operator provides: PostgreSQL (CNPG) and Redis (light), using only the
configuration the operator already injects (read contracts from specs/002-shop-operator); no
operator changes unless unavoidable and called out. Redis access MUST use authentication (deferred
finding R-3 from 002). Shop admin auth: simplest secure option, one admin per shop. Integration
tests with Testcontainers for both databases (5.2). CI: build, test, SemVer image to GHCR,
consistent with shop-operator CI. Update helm-charts/shop only if needed. Out of scope: crypto
payment and wallet checking (spec 005), per-shop observability (spec 006), ShopHub backend
(spec 004)."

## Clarifications

### Session 2026-11-23

- Q: How does the one admin get the first password? → A: The operator generates a random admin
  password per Shop in a Secret owned by the Shop and passes it to the app by secret reference
  (FR-004).
- Q: When does stock go down? → A: Immediately when the order is placed; spec 005 gives it back if
  payment never arrives (FR-014).
- Q: Does light-shop data survive a Redis restart? → A: Yes. The same operator release adds
  persistent storage for light-shop Redis (FR-021).
- Delivery order (maintainer): the operator change (Redis auth, admin Secret, Redis persistence)
  is its own first phase with its own operator release, chart release and kube-state PR,
  completed **before any app work** (Dependencies).
- Recorded deviation (maintainer, 2026-11-23): shophub-specs PR #1 (spec 002) was merged with a
  merge commit, `79cc773`, which breaks Constitution II / Requirement 5.1 linear history. It is
  accepted as a one-off; `main` is not rewritten. Merge commits are now disabled in shophub-specs
  (squash and rebase only). Carry this entry into the tasks.md implementation log.

## User Scenarios & Testing *(mandatory)*

Actors (Requirement "Uloge u sistemu", Shop part):

- **Admin**: the shop owner. Exactly one per shop. Creates, changes and deletes items and lists
  orders.
- **Customer** ("Korisnik"): an anonymous visitor of the shop. Searches items, fills a cart and
  places an order. Paying for it is spec 005.
- **Maintainer**: the solo developer. Releases the app and, until the ShopHub backend exists
  (spec 004), plays the backend by creating test Shops in a throwaway namespace (as in 002).

### User Story 1 - Admin manages the shop's items (Priority: P1)

The admin signs in to their shop and adds items with a name, the number of pieces available and
a price. They can change any of these values later and delete items they no longer sell.

**Why this priority**: Requirement 2.1. A shop without items has nothing to sell, so nothing else
in this feature can be shown without it.

**Independent Test**: Deploy one Shop, sign in as its admin, create, edit and delete items, and
confirm that the item list reflects every change and survives an application restart.

**Acceptance Scenarios**:

1. **Given** a newly deployed shop, **When** the admin signs in with the password from the
   shop's generated admin Secret (FR-004), **Then** they reach the admin area.
2. **Given** a signed-in admin, **When** they create an item "Blue T-shirt", 12 pieces, 15.00,
   **Then** the item appears in the admin item list and in the storefront.
3. **Given** an existing item, **When** the admin changes its stock to 0, **Then** the storefront
   shows it as unavailable and it can no longer be added to a cart.
4. **Given** an existing item, **When** the admin deletes it, **Then** it disappears from the
   storefront and the admin list, and orders that already contain it are unchanged.
5. **Given** a visitor who is not signed in as admin, **When** they try any admin action,
   **Then** it is refused and no data changes.
6. **Given** invalid input (empty name, negative stock, price of 0 or below, more than 2
   decimals), **When** the admin saves, **Then** it is refused with a message naming the field.

---

### User Story 2 - Customer finds items and places an order (Priority: P1)

A customer opens the shop, searches items by name, adds items to a cart, adjusts quantities and
places an order. The order is recorded with status **PendingPayment** and the customer sees an
order reference they can use later to pay (spec 005).

**Why this priority**: Requirement 2.3 plus the order that 2.2 lists and 2.4 later pays. This is
the shop's reason to exist.

**Independent Test**: With items in the shop, search, fill the cart, place an order, and confirm
the order exists with status PendingPayment, the correct lines and total, and that stock went
down by the ordered quantities.

**Acceptance Scenarios**:

1. **Given** items "Blue T-shirt" and "Red cap", **When** the customer searches "shirt",
   **Then** only "Blue T-shirt" is listed (case-insensitive match on the name).
2. **Given** an available item, **When** the customer adds 2 pieces to the cart, **Then** the cart
   shows the item, quantity 2 and the line total; the cart survives a page reload.
3. **Given** a cart, **When** the customer asks for more pieces than are in stock, **Then** the
   quantity is limited to the available stock and the customer is told why.
4. **Given** a valid cart, **When** the customer places the order, **Then** an order with status
   PendingPayment, a unique reference, the lines (name, unit price, quantity), the total and the
   creation time is stored, each item's stock goes down by the ordered quantity, the reference is
   shown and the cart is emptied.
5. **Given** a cart whose item price or stock changed since it was filled, or whose item was
   deleted, **When** the customer places the order, **Then** no order is created; the customer
   sees what changed and can confirm the corrected cart.
6. **Given** two customers ordering the last piece of an item at the same moment, **When** both
   place orders, **Then** exactly one order succeeds and the other customer is told the item is no
   longer available.

---

### User Story 3 - Admin lists orders (Priority: P2)

The signed-in admin sees all orders of their shop, newest first, with reference, creation time,
status, total and lines.

**Why this priority**: Requirement 2.2. Needed to run the shop, but only meaningful once orders
can be placed (US2).

**Independent Test**: Place three orders as a customer, sign in as admin and confirm all three are
listed newest first with correct details; confirm the list is refused without admin sign-in.

**Acceptance Scenarios**:

1. **Given** three placed orders, **When** the admin opens the orders list, **Then** all three
   appear newest first, each with reference, time, status PendingPayment, total and lines.
2. **Given** more than one page of orders, **When** the admin pages through, **Then** every order
   appears exactly once.
3. **Given** an order containing an item that was later renamed, repriced or deleted, **When** the
   admin views it, **Then** it still shows the name and price at the time of ordering.

---

### User Story 4 - The same release runs on both database modes, deployed by the operator (Priority: P2)

The maintainer tags a release. CI builds and tests it, publishes a SemVer image, and the
shop-operator deploys that image for every Shop. A `standard` (PostgreSQL) shop and a `light`
(Redis) shop behave identically, and the light shop's database refuses unauthenticated access.

**Why this priority**: The database choice is part of Requirement 1.2 and the operator contract
(002). Without this story only one mode would work, and R-3 from 002 stays open.

**Independent Test**: Run the shared integration suite against both databases in CI; on the local
cluster, create one standard and one light Shop with the released image and run the US1–US3
scenarios on each; try to connect to the light shop's database without credentials.

**Acceptance Scenarios**:

1. **Given** a pull request, **When** CI runs, **Then** build, unit tests and integration tests
   against real PostgreSQL and real Redis must pass before merge.
2. **Given** a release tag `vX.Y.Z`, **When** CI finishes, **Then** a public image tagged `X.Y.Z`
   exists in the registry and no `latest` tag is used anywhere.
3. **Given** the operator configured with that image, **When** a `standard` and a `light` Shop are
   created, **Then** both become Ready and pass the US1–US3 scenarios.
4. **Given** a `light` shop, **When** anything in the cluster connects to its database without the
   password, **Then** the connection is refused.
5. **Given** a shop with items and orders, **When** the app is upgraded to a newer release, its
   pods restart, or (light shop) its Redis pod restarts, **Then** all items and orders are still
   there.
6. **Given** a `standard` shop with 2 or 3 replicas, **When** an admin signs in and works,
   **Then** their session keeps working whichever replica serves a request.

---

### Edge Cases

- **First start on an empty database with 2–3 replicas starting at once**: exactly one data
  set-up happens; no replica fails or duplicates data.
- **Database not reachable yet** (still provisioning, restarting): the app keeps retrying, does
  not report itself ready to receive traffic before it can serve requests, and becomes usable
  within 1 minute after the database is reachable.
- **Database becomes unreachable while running**: customers and the admin get a clear "temporarily
  unavailable" message; no partial order is stored.
- **Wallet switched on the Shop** (002 supports it): orders placed afterwards record the new
  payment address; earlier orders keep the address valid when they were placed.
- **Item deleted while in carts**: placing the order shows the item as removed (US2 scenario 5).
- **Repeated failed admin sign-ins**: further attempts are slowed or blocked for a period (FR-006).
- **Very large catalog**: search stays usable up to 1,000 items per shop (SC-005).
- **Redis restart in a light shop**: data is kept on persistent storage (FR-021); the app
  reconnects with the same password and no data is lost.
- **Missing or malformed injected configuration** (no database setting): the app does not start
  and logs which setting is missing, without printing secret values.

## Requirements *(mandatory)*

### Functional Requirements

**Items (Requirement 2.1)**

- **FR-001**: The admin MUST be able to create, change and delete items. An item has a name
  (1–100 characters, unique per shop ignoring case), a stock count (whole number ≥ 0) and a price
  (> 0, at most 2 decimal places, in mUSDT per Constitution IV).
- **FR-002**: Invalid item input MUST be refused with a message naming the field (US1 scenario 6).
- **FR-003**: Deleting an item MUST NOT change existing orders (FR-016).

**Admin access (simplest secure option, one admin per shop)**

- **FR-004**: Each shop MUST have exactly one admin account, signing in with a password. The
  initial password is generated randomly per Shop by the shop-operator, kept in a Secret owned by
  the Shop (a derived resource, Constitution V) and passed to the app only by secret reference.
  The app MUST use it to set up the admin account on first start. It MUST NOT be shown in the
  Shop status, events or logs. **Operator change** — see Dependencies.
- **FR-005**: Admin passwords MUST be stored only as a salted, slow hash; the admin MUST be able
  to change their password (minimum 12 characters). A changed password is stored by the app and
  MUST NOT be overwritten by the generated initial password on later starts.
- **FR-006**: After 5 failed sign-ins within 15 minutes, further sign-in attempts MUST be refused
  for 15 minutes.
- **FR-007**: Admin sessions MUST expire after 8 hours and on sign-out, and MUST work across all
  replicas of a shop.
- **FR-008**: Every admin function (item changes, order list, password change) MUST refuse
  requests without a valid admin session. Customer functions MUST NOT require sign-in.

**Storefront and cart (Requirement 2.3)**

- **FR-009**: The storefront MUST show the shop name from the Shop resource and list items with
  name, price and availability.
- **FR-010**: Customers MUST be able to search items by a case-insensitive substring of the name.
- **FR-011**: Customers MUST be able to add items to a cart, change quantities and remove lines.
  The cart MUST persist in the customer's browser across reloads and MUST NOT require an
  account. Quantities MUST be limited to the current stock.
- **FR-012**: Items with stock 0 MUST be shown as unavailable and MUST NOT be addable to a cart.

**Orders (Requirements 2.2, 2.3; payment is spec 005)**

- **FR-013**: Placing an order MUST re-check every line against current price, stock and
  existence. If anything changed, no order is created and the customer sees the differences.
- **FR-014**: Placing an order MUST reduce each item's stock by the ordered quantity in the same
  step that stores the order, so the order and the stock change either both happen or neither
  does. Giving stock back for unpaid orders is spec 005.
- **FR-015**: Concurrent orders MUST NOT sell more pieces than are in stock at the moment of
  ordering (US2 scenario 6).
- **FR-016**: An order MUST store a unique reference, creation time, status, the lines with the
  item name and unit price at ordering time and the quantity, the total, and the shop's payment
  (wallet) address at ordering time. New orders MUST have status **PendingPayment**; this feature
  never changes it.
- **FR-017**: The customer MUST see the order reference and total after ordering.
- **FR-018**: The admin MUST be able to list all orders, newest first, 20 per page.

**Database modes and platform fit**

- **FR-019**: The same image MUST work on both database modes and select the mode only from the
  configuration the operator injects (002 contract: `DATABASE_URL` for `standard`, `REDIS_URL`
  for `light`, plus `SHOP_NAME` and `WALLET_ADDRESS`). All FRs MUST behave identically in both
  modes.
- **FR-020**: Redis access MUST be authenticated (closes 002 finding R-3): the light shop's Redis
  MUST refuse connections without a password, and the app MUST receive the password only through
  a secret reference. **This requires a shop-operator change** — see Dependencies.
- **FR-021**: A light shop's data MUST survive a restart or rescheduling of its Redis pod: the
  operator MUST provision Redis with persistent storage owned by the Shop, so deleting the Shop
  still removes it. **Operator change** — see Dependencies.
- **FR-022**: Items and orders MUST survive app pod restarts and app upgrades in both modes.
  Data set-up on first start and on upgrade MUST be automatic and safe with several replicas
  starting at once.
- **FR-023**: The app MUST run inside the container contract the operator sets (HTTP on the
  operator's port, readiness by that port accepting connections, the operator's memory limit),
  as one container image holding both the web UI and the backend.
- **FR-024**: No secret value (admin password or hash, database or Redis credentials) may appear
  in logs, error pages, the Shop resource status or any repository.

**Delivery (Requirements 5.1, 5.2)**

- **FR-025**: The `shop` repository MUST have CI consistent with shop-operator: a lint/static
  check job and a test job (unit plus Testcontainers integration tests against real PostgreSQL
  and real Redis) required on every PR, branch protection before the first merge, squash-only
  merges, and a release job that publishes a public image `ghcr.io/nikola031ue/shop:X.Y.Z` from a
  `vX.Y.Z` tag.
- **FR-026**: Rolling out a release to all shops MUST go through kube-state (the shop-operator
  release's Shop image setting), not by hand. The placeholder `helm-charts/charts/shop` chart is
  left unchanged unless planning shows it is needed.

### Key Entities

- **Shop configuration**: not stored by the app; read at start from the operator's injected
  settings (shop name, payment address, database connection).
- **Admin**: the single account of a shop. Password hash, whether the generated initial password
  was replaced, password-changed time, failed-attempt counter and lock-out time.
- **Admin credential Secret** (operator-owned): the generated initial admin password of a Shop.
- **Redis credential Secret** (operator-owned, light shops): the generated Redis password.
- **Item**: name, stock, price. Owned by the shop.
- **Cart**: lives in the customer's browser only; lines of item and quantity. Not stored by the
  backend.
- **Order**: reference, creation time, status (PendingPayment in this feature), payment address,
  total, and lines.
- **Order line**: item name and unit price as at ordering time, quantity, and a link to the item
  that may later no longer exist.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Within 2 minutes of a new Shop becoming Ready, the admin can sign in and create the
  first item that customers can see — in both database modes.
- **SC-002**: A customer can go from opening the shop to a placed order (search, add to cart,
  order) in under 1 minute, and the order is visible in the admin list within 5 seconds.
- **SC-003**: 100% of the shared integration scenarios pass against both PostgreSQL and Redis in
  CI for every merged PR.
- **SC-004**: With 5 pieces in stock and 20 simultaneous single-piece orders, no more than 5 pieces
  are ever sold.
- **SC-005**: With 1,000 items in a shop, search results appear within 1 second.
- **SC-006**: 100% of admin functions refuse unauthenticated requests; the sixth wrong password
  within 15 minutes is refused even if correct.
- **SC-007**: A light shop's database refuses 100% of connection attempts without the password;
  scans of repositories, app logs and resource status find 0 secret values.
- **SC-008**: From pushing a release tag to a public, pullable image takes under 15 minutes, and
  shops created with that image pass the US1–US3 scenarios on the local cluster.
- **SC-009**: After an app upgrade or pod restart in both modes, and after a Redis pod restart in
  a light shop, 100% of items and orders are still present.

## Assumptions

- **Placeholder for payment**: orders stay PendingPayment. Paying, wallet checking, order expiry
  and releasing stock are spec 005.
- **Customers are anonymous**: no customer accounts or personal data are collected. The order
  reference is the customer's handle for payment in spec 005.
- **Currency**: prices are in mUSDT (Constitution IV) with 2 decimal places for display and
  entry.
- **Reaching the shop**: the operator exposes each shop only inside the cluster. In this feature
  the maintainer reaches shops through port-forwarding; public exposure and "open the shop in a
  new tab" belong to spec 004.
- **TLS** is terminated by the platform in front of the shops (spec 004); this feature does not
  add certificates.
- **Observability**: per-shop metrics, logs, traces and dashboards are spec 006 (Principle VII
  deviation already recorded in 002). The app writes ordinary logs to standard output.
- **Image rollout**: the operator runs one Shop image for all shops (002 chart value
  `shopImage`), so a release reaches every shop at once through kube-state.
- **Database mode is fixed per shop** (immutable in 002), so moving data between modes is not
  needed.
- **Scale**: up to 1,000 items and 10,000 orders per shop; a handful of simultaneous customers.
- **UI language**: English.

## Dependencies

- **shop-operator 0.2.0 (unavoidable, called out)**: operator 0.1.0 provisions light-shop Redis
  without a password or storage and injects only `SHOP_NAME`, `WALLET_ADDRESS` and
  `DATABASE_URL`/`REDIS_URL` (002 resources). FR-004, FR-020 and FR-021 cannot be met by the app
  alone. Operator 0.2.0 MUST, for every Shop:
  - generate an admin password Secret and pass it to the app by secret reference (FR-004);
  - for light shops, generate a Redis password Secret, make Redis require it and pass it to the
    app by secret reference (FR-020);
  - for light shops, give Redis persistent storage (FR-021);
  - give the shop app container more memory (request 128 MiB, limit 256 MiB), so the app does not
    depend on fitting the 0.1.0 limit of 128 MiB (FR-023).

  All new Secrets and storage are derived resources owned by the Shop. Existing 0.1.0 Shops MUST
  be brought to the new state by reconciliation, without the maintainer changing them by hand.
  This is delivered as the **first phase, before any app work**: operator release `0.2.0`,
  `shop-operator` chart release, and a kube-state PR, each with its own PR and CI. The 002
  contracts are updated to the new injected settings.
- Released 002 platform: shop-operator, CNPG and the Redis operator installed via kube-state.
- GitHub repository `shop-spec` (local folder `shop`), currently containing only an initial
  commit and no branch protection.
