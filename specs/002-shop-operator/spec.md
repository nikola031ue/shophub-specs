# Feature Specification: Shop Operator (Requirements 3.1, 3.2)

**Feature Branch**: `002-shop-operator`

**Created**: 2026-11-22

**Status**: Draft

**Input**: User description: "Shop operator per Requirements 3.1 and 3.2. Three custom resources:
Shop (name, availability standard=2 replicas / high=3, database standard=PostgreSQL via CNPG /
light=Redis via its operator, wallet reference), DiscordChannel (creates a notification channel
on our Discord server and exposes its webhook for alerts), Wallet (blockchain account on Ethereum
Sepolia where shop payments arrive). Changing a Shop updates the running app (e.g. replicas,
wallet); deleting it removes everything it created. Each resource reports status (ready / error
with reason). The shop-operator chart from feature 001 gets real content: installs the operator
and the CRDs. The Shop app itself does not exist yet, so the operator deploys a configurable
placeholder image until feature 3. Keep it minimal, solo developer, max ~25 tasks."

## Clarifications

### Session 2026-11-22

- Q: Wallet account — generate or register? → A: Both (C). Address given → validated and
  registered; omitted → the operator generates a key pair once and keeps the private key only in
  a cluster secret. Minimal: no chain interaction, so no local test chain (FR-009–FR-011).
- Q: Who creates a Shop's DiscordChannel? → A: Always the backend, as a separate resource; the
  Shop has a required reference to it (FR-001, FR-016).
- Q: How are the database operators installed? → A: Own `kube-state` releases from upstream
  charts, before `shop-operator` (FR-024).
- Constitution amended to v1.2.0 (MINOR), then v1.2.1 (PATCH): operator-created resources are
  derived resources owned by the user resource they realise (Principle V).

## User Scenarios & Testing *(mandatory)*

Actors: the **ShopHub backend** (creates, changes and deletes these resources on behalf of a
ShopHub user, per constitution Principle V) and the **maintainer** (installs the platform and
watches status). In this feature there is no ShopHub backend yet, so the maintainer plays its
role in tests.

### User Story 1 - Declare a Shop and get a running shop with its database and wallet (Priority: P1)

The backend submits a Wallet, a DiscordChannel and then a Shop with a name, an availability
level, a database type and references to that Wallet and DiscordChannel. The platform starts the shop application with the right number of copies, provides
the chosen database, gives the application the wallet address where payments arrive, and reports
when the shop is ready (or why it is not). Changing the Shop changes the running shop; deleting
it removes everything that was created for it.

**Why this priority**: This is the core of Requirement 3.1 and the thing Requirement 1.2
(ShopHub shop management) builds on. Without it no shop can exist.

**Independent Test**: With the operator installed, create a Wallet, a DiscordChannel and a Shop
that references both (in tests the DiscordChannel may point at a stubbed Discord); observe the shop running with 2 copies, a database of the chosen kind, the wallet address
available to the application and a ready status. Change availability to `high`, then delete the
Shop and check nothing it created remains.

**Acceptance Scenarios**:

1. **Given** a ready Wallet, **When** a Shop with availability `standard` and database
   `standard` is submitted, **Then** the shop application runs with 2 copies, a PostgreSQL
   database exists for it, the application receives the wallet address and its database
   connection, and the Shop status becomes ready.
2. **Given** a Shop with database `light`, **When** it is submitted, **Then** a Redis database is
   provided instead of PostgreSQL and the Shop becomes ready.
3. **Given** a ready `standard` Shop, **When** availability is changed to `high`, **Then** the
   shop runs with 3 copies and stays ready.
4. **Given** a ready Shop, **When** its wallet reference is changed to another ready Wallet,
   **Then** the running application uses the new wallet address.
5. **Given** a ready Shop, **When** it is deleted, **Then** the application, its database and
   every other resource created for that Shop are removed.
6. **Given** a Shop that references a Wallet or DiscordChannel that does not exist or is not
   ready, **When** it is submitted, **Then** the Shop reports an error with a reason naming that
   resource, and becomes ready by itself once it is ready.
7. **Given** a Wallet submitted without an address, **When** it is reconciled, **Then** a new
   Sepolia account is generated, its private key is stored only in a cluster secret named in the
   status, and the status shows the address. **Given** a Wallet submitted with an address,
   **Then** the address is checked for a valid format and shown in status; no key is stored.
8. **Given** a Shop with an invalid value (e.g. availability `medium`), **When** it is
   submitted, **Then** it is rejected immediately with a message naming the field and the allowed
   values.

---

### User Story 2 - Install the operator and its resource types from kube-state (Priority: P2)

The maintainer brings up the platform from `kube-state` as in feature 001. The `shop-operator`
release now installs the real operator and the three resource types (Shop, Wallet,
DiscordChannel), so the cluster accepts these resources right after bring-up (Requirement 3.2).

**Why this priority**: Requirement 3.2 and constitution Principle V require the operator and
its resource types to be platform state installed by its chart. Story 1 can be developed and
tested without it, but it is needed to run on a real cluster.

**Independent Test**: On an empty local cluster run the feature 001 bring-up; the
`shop-operator` release becomes ready, the three resource types are registered and accepted, and
the `shophub-discord` release (which waits for these types) installs after it.

**Acceptance Scenarios**:

1. **Given** an empty local cluster, **When** the bring-up runs, **Then** the operator is
   running and Shop, Wallet and DiscordChannel resources can be submitted.
2. **Given** a new operator version, **When** the chart version in `kube-state` is bumped and
   bring-up re-runs, **Then** the operator and resource types are upgraded and existing Shops keep
   running.
3. **Given** the placeholder application image is configured in the release's values, **When** a
   Shop is created, **Then** that image (with a pinned version, never `latest`) is what runs.

---

### User Story 3 - Each Shop can alert to its own Discord channel (Priority: P3)

A DiscordChannel resource creates a notification channel on the project's Discord server and
makes the channel's webhook available inside the cluster, so alerting (a later observability
feature, Requirement 4.1) can post to it. Deleting the resource removes the channel.

**Why this priority**: Required by Requirement 3.1 and needed by the observability feature, but
shops work without it.

**Independent Test**: Submit a DiscordChannel; a channel with the expected name appears on the
Discord server, the webhook is stored in the cluster as a secret named in the resource status,
and a test message posted through it appears in the channel. Delete the resource; the channel
disappears.

**Acceptance Scenarios**:

1. **Given** the operator has access to the Discord server, **When** a DiscordChannel is
   submitted, **Then** a channel is created, its webhook is stored as a cluster secret, and the
   status becomes ready and names that secret (never the webhook value itself).
2. **Given** a ready DiscordChannel, **When** it is deleted, **Then** the channel is removed
   from the Discord server and the stored webhook is removed from the cluster.
3. **Given** the Discord server is unreachable or rejects the request, **When** a
   DiscordChannel is submitted, **Then** the status reports an error with the reason and the
   operator retries until it succeeds.
4. **Given** a Shop that references a DiscordChannel, **When** the DiscordChannel is ready,
   **Then** the Shop status shows which channel (and webhook secret) its alerts will use.

---

### Edge Cases

- **Immutable choices**: changing a Shop's database type after creation is rejected with a
  message (data migration is out of scope); the shop name is fixed after creation.
- **Wallet or channel in use**: deleting a Wallet or DiscordChannel that a Shop still
  references is held until no Shop references it; its status says which Shops use it.
- **Generated wallet deleted**: deleting a generated Wallet also deletes its private key, so any
  funds on that account become unreachable. Acceptable because only testnet tokens are used
  (Principle IV); documented in the operator README.
- **Invalid address**: a Wallet with a malformed address is rejected at submission.
- **Generated key Secret deleted by hand**: the private key can't be restored and a new key
  would silently change the payment address, so the Wallet reports an error (key lost), keeps
  showing its old address and does not generate a new key. This is the one exception to FR-018.
- **Partial failure**: if the database cannot be created (e.g. its operator is missing), the
  Shop reports an error naming the database and keeps retrying; nothing is left half-deleted when
  the Shop is later removed.
- **Manual tampering**: if something created for a Shop is changed or deleted by hand, the
  operator restores it to match the Shop.
- **Operator restart**: restarting or upgrading the operator does not recreate, duplicate or
  delete existing shops, databases, wallets or Discord channels.
- **Duplicate names**: two Shops (or DiscordChannels) with the same name in the same namespace
  are impossible; the same name in different namespaces is allowed and yields separate
  resources. Discord channel names may repeat (Discord allows it); each channel is tied to its
  resource by the resource UID in the channel topic, which keeps creation idempotent.
- **Discord rate limits**: rate-limited requests are retried later and reported in status, not
  treated as permanent failures.
- **Secrets missing**: if the Discord access secret or any secret the operator needs is missing,
  the affected resources report an error naming the missing secret (not its value).

## Requirements *(mandatory)*

### Functional Requirements

**Shop**

- **FR-001**: The platform MUST accept a Shop resource with: shop name, availability
  (`standard` | `high`), database (`standard` | `light`), and required references to a Wallet
  and a DiscordChannel in the same namespace. A Shop is ready only when both referenced
  resources are ready. Missing or invalid values MUST be rejected at submission with a message naming the
  field and allowed values.
- **FR-002**: For each Shop the operator MUST run the shop application with 2 copies for
  `standard` and 3 copies for `high` availability.
- **FR-003**: For database `standard` the operator MUST provide a PostgreSQL database managed by
  the PostgreSQL operator (CloudNativePG); for `light` a Redis database managed by a Redis
  operator. The operator MUST NOT run databases itself (Requirement 1.2 note).
- **FR-004**: The shop application MUST receive the shop name, the address of the referenced
  Wallet and the connection details of its database; credentials MUST reach it only through
  cluster secrets (Principle VI).
- **FR-005**: Changing a Shop's availability or wallet reference MUST update the running shop to
  match; changing the database type or shop name MUST be rejected.
- **FR-006**: Deleting a Shop MUST remove everything the operator created for it, including the
  database and its data.
- **FR-007**: Until feature 3 delivers the Shop application, the operator MUST run a placeholder
  application whose image and version are set in the operator's chart values; the version MUST be
  a pinned SemVer tag, never `latest`.
- **FR-008**: The shop MUST be reachable inside the cluster at an address reported in the Shop
  status, for the ShopHub backend. Browser access ("open in a new tab", Requirement 1.2) comes
  with the later feature that adds an ingress controller.

**Wallet**

- **FR-009**: The platform MUST accept a Wallet resource representing an Ethereum Sepolia account
  where a shop's payments arrive, with an optional address, and MUST report the account address
  in its status:
  - address given → the operator only validates its format (including checksum when mixed
    case) and registers it; no private key exists in the cluster;
  - address omitted → the operator generates a new account once and stores its private key
    only in a cluster secret owned by the Wallet, whose name is reported in status. The key is
    never regenerated while that secret exists. Nothing beyond key generation is in scope
    (no funding, balance checks or transactions).
- **FR-010**: A Wallet MUST NOT be removed while a Shop references it (deletion waits, status
  lists the referencing Shops). Deleting a generated Wallet deletes its key secret.
- **FR-011**: The operator MUST NOT contact any blockchain node in this feature; Wallet behaviour
  is offline (key generation and address validation), so tests need no local chain and CI never
  contacts Sepolia (Principle IV).

**DiscordChannel**

- **FR-012**: The platform MUST accept a DiscordChannel resource; the operator MUST create a
  matching text channel on the project's Discord server and a webhook for it.
- **FR-013**: The webhook MUST be stored only in a cluster secret whose name is reported in the
  DiscordChannel status; the webhook value MUST NOT appear in status, logs or events.
- **FR-014**: Deleting a DiscordChannel MUST remove the channel from the Discord server and the
  stored webhook secret.
- **FR-015**: The operator's access to the Discord server MUST come from a cluster secret
  referenced by name in the chart values, never from a value committed to git (Principle VI).
- **FR-016**: DiscordChannels MUST always be created by the backend as separate resources; the
  operator MUST NOT create DiscordChannels on its own. A Shop references exactly one; the
  referenced channel's webhook secret name MUST appear in the Shop status. A DiscordChannel MUST
  NOT be removed while a Shop references it.

**All resources**

- **FR-017**: Each Shop, Wallet and DiscordChannel MUST report status: ready, or not ready with
  a short reason code and a human-readable message, plus the last time the status changed.
- **FR-018**: The operator MUST keep everything it created in line with the resource: manual
  changes are reverted and transient failures are retried with increasing delays.
- **FR-019**: Everything the operator creates MUST carry labels identifying the owning shop and
  the owning user labels copied from the parent resource (Principle V).
- **FR-020**: The operator MUST run with the least permissions needed for the kinds it manages;
  no wildcard permissions or cluster-admin.

**Packaging and installation**

- **FR-021**: The `shop-operator` chart MUST install the operator and the Shop, Wallet and
  DiscordChannel resource types (Requirement 3.2), replacing the feature 001 placeholder, with a
  bumped SemVer chart version.
- **FR-022**: The `kube-state` `shop-operator` release MUST reference the new chart version; a
  bring-up on an empty cluster MUST end with the operator running and the resource types usable.
- **FR-023**: The operator MUST be built from its own repository (`shop-operator`), with CI that
  runs unit and integration tests and publishes a SemVer-tagged image (Principles II, III).
- **FR-024**: The database operators (CloudNativePG and a Redis operator) MUST be installed as
  their own `kube-state` releases, using their upstream charts at pinned versions, ordered before
  `shop-operator` in `releaseOrder`, so a single bring-up from `kube-state` still provides
  everything (feature 001 FR-018 behaviour, including waiting for their resource types).
- **FR-025**: The `shophub-discord` chart MUST replace its feature 001 placeholder with one
  DiscordChannel for platform and cluster alerts (feature 001 clarification Q1). This channel is
  platform state declared in git, not a user resource.

### Key Entities

- **Shop**: a user's shop. Attributes: name, availability, database type, Wallet reference,
  DiscordChannel reference, owner labels. Status: ready/error, reason, message, shop address,
  current copies, webhook secret name. Owns its application and database.
- **Wallet**: an Ethereum Sepolia account that receives a shop's payments. Attributes: optional
  address. Status: ready/error, address, key secret name (generated wallets only), Shops
  referencing it. Referenced by zero or more Shops.
- **DiscordChannel**: a notification channel on the project Discord server, created by the
  backend. Attributes: channel name. Status: ready/error, name of the secret holding the
  webhook, Shops referencing it.
- **Shop database**: the PostgreSQL or Redis instance provided for one Shop; created and removed
  with it.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: On the local cluster, a newly submitted `standard` Shop (with a ready Wallet and a
  ready DiscordChannel) reports ready within 5 minutes, including its database.
- **SC-002**: An availability change is reflected in the number of running copies within
  2 minutes.
- **SC-003**: Within 2 minutes of deleting a Shop or DiscordChannel, none of the resources (or
  the Discord channel) created for it remain.
- **SC-004**: 100% of Shops with invalid or missing fields are rejected at submission with a
  message naming the field.
- **SC-005**: Whenever a resource cannot become ready, its status shows a reason within
  1 minute, so a maintainer can diagnose it without reading operator logs.
- **SC-006**: A bring-up on an empty local cluster installs the operator and the three resource
  types in a single run, still within the feature 001 limit of 15 minutes.
- **SC-007**: No secret value (database password, Discord webhook or token, private key) appears
  in any repository, resource status, event or operator log.

## Assumptions

- Requirements 3.1 and 3.2 of `docs/ShopHub.md` are the source; Requirement 1.2 defines the
  Shop fields; Requirement 4.1 needs the per-Shop Discord channel later.
- Per-Shop dashboards, metrics and alert rules (Principle VII) are out of scope; this feature
  only provides the Discord channel and labels they will use.
- The ShopHub backend, its ServiceAccount and RBAC, and the real Shop application are later
  features; here resources are submitted by the maintainer in tests.
- A Shop and everything created for it live in the Shop's namespace; namespace-per-user
  layout is decided by the ShopHub feature.
- Payment token handling (mUSDT contract, checking incoming payments) is out of scope; the
  Wallet only provides the account where payments arrive.
- One project Discord server exists and its access credential is provided by the maintainer as
  a cluster secret.
- Shop exposure on the local cluster uses whatever the local cluster offers (detailed in the
  plan); a public domain/TLS is out of scope.
- **Constitution v1.2.1** (v1.2.0 MINOR 2026-11-22, v1.2.1 PATCH 2026-11-23): Principle V
  separates user resources (Shop, Wallet, DiscordChannel — backend only) from derived resources
  (databases, workloads, secrets — shop-operator only, owned by the user resource they realise).
  This feature follows it.
- Per-Shop observability (Principle VII) is deferred to the observability feature; the deviation
  is recorded in plan.md Complexity Tracking.
- The `kube-state` schema check from feature 001 only allows charts from the project registry;
  it must be extended to allow the pinned upstream database-operator charts (plan decides how).
- Generated Wallet keys are acceptable only because payments use testnet tokens (Principle IV).
