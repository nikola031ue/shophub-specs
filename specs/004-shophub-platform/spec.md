# Feature Specification: ShopHub Platform (Requirements 1.1, 1.2, 3.3)

**Feature Branch**: `004-shophub-platform`

**Created**: 2026-11-24

**Status**: Draft

**Input**: User description: "ShopHub platform (requirements 1.1, 1.2, 3.3 from ShopHub.md): Angular
+ .NET web app where a registered user signs in and creates, configures (availability standard=2/
high=3 replicas, wallet, database standard/light), opens in a new tab, and deletes shops. The
backend is the only writer of user resources (Shop, Wallet, DiscordChannel) via the Kubernetes
API, per constitution Principle V; use only the shop-operator 0.2.1 CRDs, change them only if
unavoidable and called out. Phase 1 before any app work: ingress controller in kube-state,
per-shop Ingress created by the operator so each shop has a URL, trusted forwarded headers
(deferred D-1 from 003: per-IP lockout, per-client rate limit, Secure cookie), and narrow the
operator's cluster-wide Secret access (deferred R-1 from 002). Clarify how requirement 1.2 "admin
sets wallet address" maps to the Wallet resource from 002. Web3 login is optional, out of scope.
ShopHub helm chart (3.3) and kube-state rollout. Out of scope: payments (005), observability
(006)."

## Clarifications

### Session 2026-11-24

- Q: How does the owner's wallet address (Requirement 1.2) map to the 002 Wallet? → A: Always a
  **registered** Wallet holding the owner's own address. ShopHub and the platform never hold
  keys. A Wallet's address is immutable, so changing the address means a new Wallet and a Shop
  pointing at it; the old Wallet is removed once unused (FR-009).
- Q: How does the owner get the shop admin's initial password? → A: The panel **shows the
  operator-generated password** (`<shop>-admin`) to the owner on request, until the shop admin
  has changed it. CRDs unchanged (FR-016).
- Q: Where do shops live? → A: **One shared namespace for all shops.** ShopHub and the
  shop-operator each get only a namespaced Role there, which closes 002 R-1. ShopHub's Secret
  read is limited to that namespace and used only for `<shop>-admin`. Owner isolation is
  enforced in the backend and covered by tests: owner A cannot see, change or delete owner B's
  shops (FR-005, FR-018).

## User Scenarios & Testing *(mandatory)*

Actors (Requirement "Uloge u sistemu", ShopHub part):

- **Owner** ("Korisnik" of ShopHub, called "admin" in Requirement 1.2): a registered ShopHub user
  who creates, configures, opens and deletes their own shops. Not to be confused with a shop's
  own **shop admin** from spec 003, who manages that shop's items and orders.
- **Customer**: an anonymous visitor of a shop (spec 003). Affected here only through the shop's
  public address and the protections in User Story 1.
- **Maintainer**: the solo developer. Runs the platform through `kube-state` and releases.

Delivery order (maintainer): User Story 1 is **Phase 1**. It is released and deployed through
`kube-state` before any ShopHub application work starts (Dependencies).

### User Story 1 - Every shop has its own secure address (Priority: P1, Phase 1)

Each shop is reachable from a browser at its own HTTPS address. ShopHub itself also has its own
HTTPS address. Because traffic now arrives through a shared entry point, the shop app learns each
visitor's real address from that entry point, and only from it. It uses that address, together with the
account, to lock out attackers instead of locking out the whole shop, and to limit orders per customer
instead of per entry point. Session cookies are sent over HTTPS only. In the same phase, the
shop-operator loses its ability to read Secrets everywhere in the cluster.

**Why this priority**: "Open the shop in a new tab" (Requirement 1.2) needs a real address per
shop. The deferred 003 findings (D-1) become real problems the moment a shared entry point
exists, so they are fixed before it is used. The deferred 002 finding R-1 (cluster-wide Secret
access) is a least-privilege gap (Principle V) that grows with every shop.

**Independent Test**: With the platform deployed from `kube-state`, the maintainer creates two
test Shops in the shops namespace (as in 002/003). Each is reachable at its own HTTPS address. Failed sign-ins for an account from one
client address do not block the right password from another address. Two customers on different
addresses can each place orders up to the limit. The operator's identity is refused when it tries
to read a Secret outside the namespaces it is allowed.

**Acceptance Scenarios**:

1. **Given** a Shop that is Ready, **When** a browser opens the address shown in its status,
   **Then** the storefront loads over HTTPS. The same address over plain HTTP redirects to
   HTTPS.
2. **Given** two Ready Shops, **When** their addresses are compared, **Then** they differ, and
   each serves only its own shop.
3. **Given** 5 failed shop-admin sign-ins from client address A, **When** the admin signs in with
   the right password from client address B, **Then** sign-in succeeds, and address A stays
   locked for 15 minutes.
4. **Given** a client that sends its own forwarded-address header directly, **When** it tries to
   appear as another address, **Then** the shop ignores that header and uses the address
   reported by the platform entry point.
5. **Given** customer A has used up the order limit, **When** customer B on another address
   places an order, **Then** it is accepted.
6. **Given** a shop-admin sign-in over HTTPS, **When** the session cookie is inspected, **Then**
   it is marked Secure, HttpOnly and SameSite=Strict.
7. **Given** the operator's identity, **When** it asks to read Secrets in any namespace other
   than the shops namespace (apart from the one Discord token Secret in its own namespace),
   **Then** the request is refused. Shops in the shops namespace keep working.

---

### User Story 2 - A person registers and signs in to ShopHub (Priority: P1)

A visitor creates a ShopHub account with an email address and a password, then signs in to reach
their shop panel. Signing out ends the session.

**Why this priority**: Requirement 1.1. Every other ShopHub function sits behind it.

**Independent Test**: Register a new account, sign out, sign back in, and reach an empty shop
panel. A second account cannot see the first one's panel.

**Acceptance Scenarios**:

1. **Given** no account for an email, **When** the visitor registers with that email and a valid
   password, **Then** the account exists, and the panel signs them in with those credentials to an
   empty shop panel.
2. **Given** an existing account, **When** someone registers again with the same email in any
   letter case, **Then** the registration response is the same as for a new account (no account
   is created or changed), and signing in with the new password fails with the generic error.
3. **Given** a registered owner, **When** they sign in with the right password, **Then** they
   reach their shop panel. With a wrong password they get a generic error.
4. **Given** 5 failed sign-ins for one account from one client address within 15 minutes,
   **When** a 6th attempt comes from that address, **Then** it is refused for 15 minutes, even
   with the right password. Sign-in to that account from another address still works.
5. **Given** a signed-in owner, **When** they sign out, **Then** the panel is no longer reachable
   with that session, including with a copy of the session cookie.
6. **Given** a visitor who is not signed in, **When** they open any panel page or panel API,
   **Then** they are sent to sign-in (pages) or refused (API).

---

### User Story 3 - The owner creates a shop and opens it in a new tab (Priority: P1)

The signed-in owner creates a shop by entering its name, choosing availability (standard or
high), entering the wallet address that receives customer payments, and choosing the database
(standard or light). The panel shows the new shop as being prepared, then ready, together with
its address. Clicking the shop opens the storefront in a new browser tab. The owner also gets
what they need to sign in as the shop's admin.

**Why this priority**: Requirement 1.2, and the reason ShopHub exists. Together with US1 and US2
it is the MVP.

**Independent Test**: Sign in, create one standard and one light shop, wait until both show as
ready, open each in a new tab, and sign in to each shop's admin area.

**Acceptance Scenarios**:

1. **Given** a signed-in owner, **When** they create a shop with a valid name, availability,
   wallet address and database, **Then** it appears in their list as "being prepared" within 5
   seconds, and as "ready" with its address once the platform reports it ready.
2. **Given** a ready shop, **When** the owner clicks it, **Then** the storefront opens in a new
   tab at the shop's address, and the panel stays open in the original tab.
3. **Given** a ready shop, **When** the owner follows the panel's steps to reach the shop's
   admin area, **Then** they can sign in as shop admin (FR-016).
4. **Given** invalid input (empty or too long name, a duplicate name among the owner's shops, an
   unknown availability or database, a malformed wallet address or one with a bad checksum),
   **When** the owner submits, **Then** nothing is created and each problem is shown next to its
   field.
5. **Given** a shop that cannot become ready (for example the Discord channel can't be created),
   **When** the owner looks at the list, **Then** the shop shows a plain-language reason, not a
   raw platform message.
6. **Given** two owners, **When** each lists shops, **Then** each sees only their own. Using
   another owner's shop identifier in any panel API (view, change, delete, show admin password)
   gets "not found".

---

### User Story 4 - The owner changes a shop's configuration (Priority: P2)

The owner changes a shop's availability between standard (2 copies) and high (3 copies) and
changes the wallet address that receives payments. Name and database are shown but can't be
changed after creation.

**Why this priority**: Requirement 1.2 names configuration changes explicitly. Shops are usable
without them, so they come after creation.

**Independent Test**: Change a shop from standard to high and back, and change its wallet
address. Confirm the running shop follows each change with its data intact.

**Acceptance Scenarios**:

1. **Given** a ready standard shop, **When** the owner switches it to high, **Then** within 2
   minutes the shop runs 3 copies and the panel shows high. Items and orders are unchanged.
2. **Given** a ready shop, **When** the owner enters a new valid wallet address, **Then** orders
   placed after the change carry the new address, existing orders keep the address they were
   placed with, and the panel shows the new address.
3. **Given** a shop's edit form, **When** the owner looks at name and database, **Then** they are
   read-only, with a short note why (renaming would change the address; switching the database
   would lose the shop's data).
4. **Given** an invalid new wallet address, **When** the owner saves, **Then** nothing changes
   and the field shows the error.

---

### User Story 5 - The owner deletes a shop (Priority: P2)

The owner deletes a shop they no longer want. Because all of the shop's items, orders and data
are lost, the owner must confirm by typing the shop's name.

**Why this priority**: Requirement 1.2. Needed for a complete lifecycle, but less frequent than
creation and changes.

**Independent Test**: Delete a ready shop and a shop that is still being prepared. Both disappear
from the list, their addresses stop answering, and nothing of theirs is left in the cluster.

**Acceptance Scenarios**:

1. **Given** a shop, **When** the owner confirms deletion by typing its name, **Then** it shows
   as "being deleted" immediately and disappears from the list within 2 minutes. Its address no
   longer serves the shop, and its database, notification channel and wallet record are gone.
2. **Given** a mistyped confirmation, **When** the owner confirms, **Then** nothing is deleted.
3. **Given** a shop still being prepared, **When** the owner deletes it, **Then** it is removed
   just like a ready one.

---

### Edge Cases

- **Two requests for the same name at once** from one owner (double click): exactly one shop is
  created.
- **Kubernetes API unavailable or slow** during create, change or delete: the owner gets a clear
  "try again" message. Nothing is left half-created: a create that fails part-way removes what
  it made or finishes it on the next attempt, and never leaves an orphaned wallet or channel
  without a shop.
- **User resources changed or deleted outside ShopHub** (by mistake): ShopHub's record of each
  shop is authoritative. Missing or drifted Shop, Wallet and DiscordChannel resources are put
  back (Principle V: restoring runtime state is the backend's job). Shops ShopHub has no record
  of are never touched. **Limit (validated in T025):** putting back a deleted Shop restores the
  resource, not its data. The operator removes the shop's database, admin password and keys with
  the Shop, so the shop comes back empty (plan.md D-4, spec 005).
- **Shop stuck while being prepared** (database operator missing, cluster out of capacity,
  Discord unavailable): after 10 minutes, the panel shows that preparation is taking unusually
  long, plus the plain reason. The owner can still delete the shop.
- **Wallet address change while orders are being placed**: each order carries exactly one of
  the two addresses, never a mix. Payments for orders placed before the change are spec 005.
- **Owner reaches the shop limit** (FR-011): creation is refused with a message naming the limit.
- **Shop names that differ only in letter case or spaces**: treated as duplicates for that owner.
  Different owners may use the same name; the shop addresses still differ.
- **Very long or unusual names** (emoji, other alphabets): accepted as the display name within
  1–100 characters. The address uses a separate generated identifier, so the name never has to
  fit address rules.
- **Shop deleted in one tab while open in another**: the other tab's next action reports that the
  shop no longer exists.
- **A Shop created in another namespace** (by mistake, by hand): not reconciled by the operator
  and not shown in ShopHub.
- **Owner signed in on two devices**: both sessions work. Sign-out ends only that session, unlike
  the shop admin (spec 003).
- **Forwarded headers on a request that did not come through the platform entry point** (for
  example from another pod inside the cluster): ignored. The direct peer address is used.

## Requirements *(mandatory)*

### Functional Requirements

**Accounts (Requirement 1.1)**

- **FR-001**: Visitors MUST be able to register with an email address and a password of at least
  12 characters. Emails are unique regardless of letter case. No email verification is sent
  (Assumptions). No account enumeration (decided 2026-11-25): registering with an email that
  already has an account MUST give the same response, with similar timing, as registering a new
  one; signing in with an unknown email MUST give the same response, with similar timing, as a
  wrong password.
- **FR-002**: Registered owners MUST be able to sign in and sign out. A session lasts at most 8
  hours. Sign-out ends that session, and a copied cookie stops working. Changing the password
  MUST end every other session of that owner, keeping only the session that made the change
  (decided 2026-11-25); a stolen cookie of another session stops working.
- **FR-003**: After 5 failed sign-ins for one account from one client address within 15
  minutes, further sign-ins for that account from that address MUST be refused for 15 minutes,
  even with the right password. An account-wide ceiling MUST also apply (decided 2026-11-25):
  50 failed sign-ins for one account across all addresses within 15 minutes lock that account
  from every address for 15 minutes. Other accounts are never affected, and attempts made while
  a lock applies don't count, so one address alone can't trigger the account-wide lock.
- **FR-004**: Passwords MUST be stored only as salted, slow hashes. Session cookies MUST be
  Secure, HttpOnly and SameSite=Strict, and state-changing requests MUST carry an anti-forgery
  token.
- **FR-005**: Every panel page and panel API MUST require a signed-in owner. Each owner MUST be
  able to see, change and delete only their own shops. A request naming another owner's shop
  MUST get the same answer as a shop that does not exist.

**Shops (Requirement 1.2)**

- **FR-006**: Owners MUST be able to create a shop by entering a display name (1–100
  characters, unique per owner ignoring letter case and surrounding spaces), availability
  (`standard` = 2 copies, `high` = 3 copies), a wallet address (`0x` + 40 hex digits; a
  mixed-case address must pass its checksum) and a database (`standard` = PostgreSQL, `light` =
  Redis).
- **FR-007**: Creating a shop MUST make ShopHub create, through the Kubernetes API, the user
  resources the shop-operator 0.2.1 CRDs need: one Shop, the Wallet it references (FR-009) and a
  DiscordChannel for its notifications. No user input is needed for the channel; its name is
  derived from the shop's generated identifier. The CRDs are used unchanged (Dependencies).
- **FR-008**: The shop list MUST show, for each of the owner's shops: display name, state
  (being prepared, ready, needs attention, being deleted), a plain-language reason when not
  ready, the address, availability, database and wallet address. The list refreshes on its own
  at least every 10 seconds while any shop is not ready.
- **FR-009**: **Wallet mapping** (Requirement 1.2 "adresu wallet-a"): each shop's Wallet MUST be
  a **registered** Wallet (002 `spec.address` set) holding the owner's own address. ShopHub MUST
  NOT create generated Wallets, and the platform holds no keys for shops. Because a Wallet's
  address is immutable, changing a shop's wallet address MUST create a new registered Wallet,
  point the Shop at it, and then delete the old Wallet. The old Wallet's deletion waits until no
  Shop uses it (002 `InUse`). **Limits:** it doesn't wait for unpaid orders, or for orders
  taken by old copies while the change rolls out, that still carry the old address (plan.md D-1,
  spec 005). Wallet creates and deletes aren't logged, so the swap order can't be traced from logs
  (plan.md D-3, spec 006).
- **FR-010**: Clicking a ready shop MUST open its storefront in a new browser tab at the shop's
  address. The link MUST NOT give the new tab access to the panel window.
- **FR-011**: An owner MUST NOT have more than 5 shops at a time (including shops being
  prepared or deleted). The limit is a platform setting.
- **FR-012**: Owners MUST be able to change a shop's availability and wallet address. Display
  name and database MUST be read-only after creation (both are immutable in the CRDs; switching
  the database would lose data). A changed wallet address applies to orders placed afterwards.
- **FR-013**: Owners MUST be able to delete a shop after typing its display name to confirm.
  Deleting MUST remove the shop's Shop, Wallet and DiscordChannel resources. Through the
  operator, that removes its app, database and data, address and notification channel. The shop
  leaves the list within 2 minutes.
- **FR-014**: Create, change and delete MUST be safe to repeat. A retried request, a double
  click or a backend restart MUST NOT create duplicates or leave a Wallet or DiscordChannel
  without its Shop.
- **FR-015**: ShopHub MUST keep its own record of each shop's owner and desired configuration.
  It MUST put back missing or drifted user resources of shops it knows. It MUST NOT touch
  resources it has no record of. (Resources only: a deleted Shop's data is not restored, see the
  edge case above and plan.md D-4.)
- **FR-016**: **Shop admin hand-off**: the owner MUST be able to reach their shop's admin area
  and sign in as its shop admin (spec 003: one admin per shop, initial password generated by the
  operator into `<shop>-admin`).
  - The panel MUST show that generated password to the shop's owner only, and only on an explicit
    request ("show shop admin password"). It is never in list responses, logs or caches.
  - ShopHub MUST read only the `<shop>-admin` Secret of a shop the requesting owner owns. The
    panel tells the owner to change the password at the first shop sign-in (spec 003 already
    asks).
  - Once the shop admin has changed the password, the generated value no longer works. The panel
    then says so instead of offering a password that doesn't work. How ShopHub learns this is a
    plan decision; it MUST NOT need a CRD change.

**Isolation and least privilege (Principle V)**

- **FR-017**: ShopHub MUST be the only writer of Shop, Wallet and DiscordChannel resources. It
  MUST act under its own identity, bound by **one namespaced Role in the shops namespace**:
  - only the verbs it needs on exactly those three kinds;
  - read-only Secret access, used only for `<shop>-admin` (FR-016).
  It MUST have no cluster-wide permissions and no access to other namespaces' resources. It MUST
  NOT create or change derived resources (databases, workloads, Secrets created by the operator).
- **FR-018**: **Where shops live**: all shops' user resources (and through the operator their
  derived resources) MUST live in **one shared shops namespace**, created as platform state in
  `kube-state`. Owner isolation is not provided by Kubernetes, so it MUST be enforced in the
  ShopHub backend on every read, change and delete (FR-005). Automated tests MUST prove that
  owner A cannot see, change, delete or read the shop admin password of owner B's shops, through
  every panel API. Generated identifiers MUST be unique within the namespace, so two owners'
  shops never collide.
- **FR-019**: Every user resource ShopHub creates MUST carry the owner and shop labels from the
  002 contract (`shophub.io/owner`, `shophub.io/shop`). The owner label MUST be a stable
  identifier, never the email address.

**Shop addresses and real-client protections (Phase 1; 003 D-1, 002 R-1)**

- **FR-020**: The platform MUST include a cluster entry point, recorded in `kube-state`, that
  routes HTTPS traffic to ShopHub and to every shop by host name. Plain HTTP MUST redirect to
  HTTPS. Locally, the certificate MUST be one the maintainer's browser trusts after a one-time
  setup.
- **FR-021**: The shop-operator MUST give every Shop its own address by creating routing for it
  as an owned child of the Shop. The address MUST be derived from the Shop's identity and a
  platform domain setting, appear in the Shop's status, and stop answering when the Shop is
  deleted. The CRD schemas MUST NOT change for this. The one allowed CRD change is a validation
  rule (decided 2026-11-25): a Shop name MUST NOT contain a dot, so its address stays a single
  label under the platform domain and is covered by the wildcard certificate.
- **FR-022**: The shop app MUST take the client address from forwarded headers only when the
  request comes from the platform entry point. Otherwise it uses the direct peer address.
- **FR-023**: The shop app's admin lockout MUST apply per (account, client address): 5 failed
  sign-ins for an account from one address in 15 minutes lock that account from that address for
  15 minutes; other accounts and other addresses are not locked. The shop app has a single admin
  account today, so the account part is fixed, but the key includes it. Rate limits stay per client
  address (FR-024). A shop-wide ceiling MUST remain against distributed
  guessing: 50 failed sign-ins across all addresses in 15 minutes lock the whole shop for 15
  minutes.
- **FR-024**: The shop app's order limit MUST be per real client address (10 orders per minute by
  default, as in 003).
- **FR-025**: The shop app's session cookie MUST always be marked Secure.
- **FR-026**: The shop-operator's cluster-wide permissions MUST be replaced by **a namespaced Role
  in the shops namespace** covering the kinds it manages there, Secrets included (closes 002
  R-1). Outside that namespace it MAY keep only what it needs in its own namespace: leader
  election, reading the Discord token Secret, and managing the platform alerts DiscordChannel
  and its webhook Secret, which move into the operator's namespace. It MUST reconcile Shops and
  Wallets only in the shops namespace, and DiscordChannels only there and in its own namespace.
  Resources in any other namespace are not reconciled.

**ShopHub chart and rollout (Requirement 3.3; Requirements 5.1–5.3)**

- **FR-027**: The `shophub` chart in `helm-charts` MUST replace today's placeholder. It deploys
  the ShopHub frontend and backend, the backend's identity and permissions (FR-017), its
  address on the platform entry point, and ShopHub's own database through the database operator
  already on the platform. It depends on the shop-operator chart's CRDs. The chart layout follows
  Requirement 5.3 (Principle I).
- **FR-028**: The ShopHub release MUST be rolled out through `kube-state` (`clusters/local/shophub`).
  Bring-up from an empty cluster MUST produce a working ShopHub at its address with no manual
  step other than the one-time local certificate trust (FR-020) and secrets created by hand as
  already documented (for example the Discord token).
- **FR-029**: The ShopHub application MUST live in its own repository with branch protection,
  required CI (build, lint, unit tests, Testcontainers integration tests against a real
  database, and tests of the Kubernetes calls against a real API server), and SemVer images
  published to GHCR from `vX.Y.Z` tags on `main`, never `latest`.
- **FR-030**: No secret (owner passwords, session keys, database credentials, shop admin
  passwords, private keys, Discord token) may appear in any repository, log, resource status or
  API response, except the deliberate hand-off in FR-016.

### Key Entities

- **Shops namespace**: the one namespace holding every shop's resources (platform state).
- **Owner account**: a registered ShopHub user. Email (unique, case-insensitive), password hash,
  stable identifier used as the owner label, created time, failed sign-in records per client
  address.
- **Owner session**: a signed-in browser of an owner, up to 8 hours; ended by sign-out.
- **Shop record**: ShopHub's own record of a shop. Owner, display name, generated identifier
  (address-safe, unique on the platform), availability, database, wallet address, desired state
  (present or being deleted), created and changed times. Source of truth for which user
  resources must exist (FR-015).
- **Shop / Wallet / DiscordChannel** (002 CRDs, unchanged): the user resources ShopHub writes for
  each Shop record. Their status (Ready and reason, address) is read back and shown to the owner
  in plain language.
- **Shop address**: `https://<generated identifier>.<platform domain>`, created by the operator
  and reported in the Shop's status.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A new visitor can register, create a standard shop and open its storefront in a new
  tab within 5 minutes on the local cluster, most of it waiting for the database. A light shop
  takes no longer.
- **SC-002**: A created shop appears in the owner's list within 5 seconds of submitting. A
  configuration change shows in the running shop within 2 minutes: the copy count, and the wallet
  address on new orders.
- **SC-003**: A deleted shop leaves the list and the cluster (app, database, address, channel,
  wallet record) within 2 minutes.
- **SC-004**: 100% of attempts to read, change or delete another owner's shop, or show its admin
  password, are refused and look the same as a shop that does not exist. Checked by automated
  tests for every panel API.
- **SC-005**: Failed shop-admin or owner sign-ins from one client address never stop the right
  password from another address (below the shop-wide ceiling). Each of two customers on different
  addresses can place 10 orders a minute.
- **SC-006**: 100% of ShopHub and shop pages and APIs reached through the platform are served over
  HTTPS. Every session cookie is marked Secure.
- **SC-007**: The operator's and ShopHub's identities are refused for every Secret read outside
  the shops namespace (the operator's one Discord token Secret excepted) and for every cluster-wide
  permission. ShopHub's identity is also refused when it tries to create or change databases,
  workloads or any Secret.
- **SC-008**: Scans of all repositories, ShopHub and shop logs, resource status and API responses
  find 0 secret values, apart from the documented FR-016 hand-off.
- **SC-009**: From an empty local cluster, bring-up from `kube-state` gives a working ShopHub at
  its address in under 15 minutes. From pushing a ShopHub release tag to a pullable image takes
  under 15 minutes.
- **SC-010**: Repeating a create (double submit or retry) never produces a second shop, and a
  create interrupted mid-way never leaves a wallet or channel without its shop. Checked by tests
  that interrupt each step.

## Validation (T025, 2026-11-26)

What the live run on an empty Docker Desktop cluster proved, and where it stops (details in
tasks.md T025):

- **SC-001:** met on platform latency, by the maintainer's decision. Create to Ready took 37 s
  (standard) and 23 s (light); registration to first storefront open was only bounded (4 min
  10 s – 5 min 54 s, derived), most of it the maintainer's idle time before creating.
- **SC-002:** listed at once; Ready in 23–37 s; high reached 3/3 in 11 s; after a wallet change a
  new order carried the new address and the earlier order kept the old one. During the rollout
  an old copy still served for a few seconds while the Shop already reported Ready (plan.md D-1).
- **SC-003:** ≤ 5.1 s and ≤ 5.5 s through the panel; 5.5 s for five API deletes; nothing left.
- **SC-004, SC-005, SC-007:** passed (isolation 404 on every endpoint; per-address lockout; the
  ServiceAccount's permissions). SC-005's order limits were validated live in Phase 1 (T010).
- **SC-006:** HTTPS with a trusted certificate (T025); the plain-HTTP redirect and `Secure`
  cookies were checked in T024 and T010.
- **SC-008:** 0 matches for 9 secret values in all repositories' history, ShopHub, operator and
  Traefik logs, resource YAML and API responses. Not covered: shop logs, shop admin and Redis
  passwords (the shops had been deleted before the scan).
- **SC-009:** bring-up 2 min 45 s; tag to pullable image 2 min 42 s.
- **Not proved:** the time order of a wallet swap (`-w2` Ready before `-w1` deleted) was seen only
  as an end state (plan.md D-3); a deleted Shop is restored without its data (plan.md D-4).

## Assumptions

- **Accounts**: email and password only. Web3 wallet sign-in (Requirement 1.1 "Opciono") is out
  of scope. No email is sent: no verification, no password reset by email. A forgotten password
  needs the maintainer, who can delete the account. Owners can't delete their own account in this
  feature.
- **One owner per shop**: no sharing of shops between owners.
- **Shop limit** of 5 per owner protects the single local cluster; it is a setting.
- **Notifications**: every shop gets its own Discord channel automatically, named after the shop's
  generated identifier. Owners do not choose or see Discord settings in this feature. Alert
  content is spec 006.
- **Local domain and certificates**: the platform domain is a local development domain that
  resolves to the local machine. The maintainer trusts a local certificate authority once.
  Public DNS and public certificates are out of scope.
- **Requirement 3.3 and prometheus-stack**: the chart's use of the prometheus-stack (dashboards,
  alerts, per-shop observability) is delivered with spec 006. This feature leaves the chart
  ready to add it and records the gap (Constitution VII, as in 002 and 003).
- **Payments** (spec 005) and **observability** (spec 006) are out of scope. Spec 005 still must
  release the stock of unpaid orders (003 dependency).
- **Hand changes**: until ShopHub's own tests can create shops, the maintainer may create test
  resources in a throwaway namespace for Phase 1 checks, deleted afterwards (as in 002 and 003).
- The single local cluster is Docker Desktop Kubernetes (constitution).

## Dependencies

- **Phase 1 platform changes (before any ShopHub app work)**. Each has its own PR, CI, release
  and `kube-state` PR:
  - a cluster entry point (ingress controller) and local certificates, as new releases in
    `kube-state` (FR-020);
  - the shared shops namespace as platform state (FR-018);
  - **shop-operator 0.3.0**: per-Shop routing and address (FR-021), plus a namespaced Role in the
    shops namespace instead of the ClusterRole (FR-026, 002 R-1). CRD schemas unchanged; one
    validation rule rejects dotted Shop names (FR-021);
  - **shop 0.2.0**: trusted forwarded headers, per-(account, address) lockout with a shop-wide ceiling,
    per-client order limit, always-Secure cookie (FR-022–FR-025, 003 D-1);
  - chart and `kube-state` updates for each.
- **shop-operator 0.2.1 CRDs** (`shophub.io/v1alpha1` Shop, Wallet, DiscordChannel) are used
  unchanged. Any change found unavoidable MUST be called out in the plan and delivered in Phase 1
  as its own operator release.
- Released 002/003 platform: shop-operator, CNPG, Redis operator, shop app 0.1.1, and the Discord
  token Secret for DiscordChannels.
- GitHub repository `shophub-spec` (local folder `shophub`), which has only an initial commit and
  no branch protection yet. It is protected like `shop-spec` before its first merge.
