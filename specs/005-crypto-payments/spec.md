# Feature Specification: Crypto Payments (Requirement 2.4)

**Feature Branch**: `005-crypto-payments`

**Created**: 2026-11-27

**Status**: Draft

**Input**: User description: "Spec 005: crypto payment on Ethereum Sepolia with a mock ERC20 mUSDT
(ETH only for gas). Buyer pays an order via MetaMask; the platform detects the Transfer to the
shop's Wallet address and marks the order paid; unpaid orders expire and return their stock. Must
include plan.md 004 deferred items D-1, D-4, D-5, D-6. Keep it minimal, solo developer, max ~25
tasks."

Builds on spec 003 (orders are placed in status PendingPayment and take their stock at once; the
order reference is the buyer's handle for paying) and spec 004 (each shop's Wallet holds the
owner's own address; the platform holds no keys). Constitution Principle IV fixes the chain
(Ethereum Sepolia), the wallet (MetaMask) and the token (the project's own test token, mUSDT), and
requires every automated test to run against a local chain, never Sepolia.

## Clarifications

### Session 2026-11-27

- Q: How is a payment matched to its order? → A: By the **paying account plus the exact total**.
  The buyer connects MetaMask at checkout; the order records that account. A settled transfer of
  exactly the total, from that account, to the order's address, after the order was placed, pays
  it. Added rule: **an account may have at most one order waiting for payment per shop**; a second
  checkout with the same account is refused until the first is paid or expired, so a transfer can
  never fit two orders.
- Q: What happens to a payment that settles after its order expired? → A: **Recorded only.** The
  order stays Expired, the transfer is listed to the shop admin as not matched, and the owner
  settles it with the buyer outside the platform. Stock is not taken again.
- Q: How should buyers get test mUSDT to pay with? → A: **Open mint on the token**: anyone can
  mint mUSDT directly from the token contract, capped per call (1,000 mUSDT), linked from the order
  page as "Get test mUSDT". The platform runs no faucet and funds nothing.
- Q: Who deploys the mUSDT token to Sepolia, and where is its address configured? → A: **The
  maintainer, once, by hand, with Remix and their own MetaMask account**; the deploying key is
  never stored anywhere. The token's address is a **public platform setting in kube-state** that
  reaches every shop; it is not a Secret.
- Q: Which Sepolia connection does the platform use to read the chain, and how is it configured?
  → A: **A provider endpoint whose URL carries an API key** (for example Infura or Alchemy). The
  full URL is a **hand-made Secret**, created once per cluster like the Discord token: never in
  git, never in logs, status or responses. Notes for the plan (maintainer): catching up must read
  the chain in small block ranges, because free provider plans limit how many blocks one query
  may cover; and whatever watches for payments needs an explicit egress allowance to the
  provider. (004's NetworkPolicies restrict only incoming traffic today, so nothing blocks it
  yet; the allowance is to be planned explicitly so a later egress restriction can't cut off
  payment detection.)
- Q: How many blocks after the one holding a payment must pass before it counts as settled? →
  A: **6 blocks** (about 72 s on Sepolia), a platform setting; the platform checks the chain for
  new blocks **at least every 12 s**.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Buyer pays an order with MetaMask (Priority: P1)

A buyer checks out in a shop (spec 003). Checkout now asks them to connect MetaMask; the order is
placed for that account and they see its reference, total and status "waiting for payment". On
the order's page they choose "Pay with MetaMask". The page asks MetaMask to switch to Sepolia if
needed and proposes a transfer of exactly the order total in mUSDT, from the connected account, to
the shop's wallet address. The buyer confirms in MetaMask. The page shows that the
payment was sent, then, once the platform has seen it settle on the chain, shows the order as
**Paid**. Only gas is paid in ETH; the price is paid in mUSDT.

**Why this priority**: Paying is the point of Requirement 2.4; without it no shop can sell.

**Independent Test**: On a local chain with mUSDT deployed, place an order, pay it from a test
account through the order page, and check that the order becomes Paid, shows the payment, and
that the shop's wallet address received exactly the total.

**Acceptance Scenarios**:

1. **Given** an order waiting for payment, **When** the buyer pays exactly its total in mUSDT to
   the shop's address from MetaMask, **Then** within 2 minutes the order page and the shop admin's
   order list show it as Paid, with the payment's transaction and the paying account.
2. **Given** MetaMask on another network, **When** the buyer chooses to pay, **Then** the page
   asks MetaMask to switch to Sepolia and sends nothing until it has.
3. **Given** a buyer without enough mUSDT, or without ETH for gas, **When** they try to pay,
   **Then** the page says which one is missing and nothing is sent.
4. **Given** a buyer who closes the tab right after confirming in MetaMask, **When** the payment
   settles, **Then** the order still becomes Paid, and reopening the order by its reference shows
   it.
5. **Given** a transfer of the wrong amount, of another token, to another address, or from an
   account other than the order's, **When** it settles, **Then** the order stays waiting for
   payment and the transfer, if it reached the shop's address, is listed to the admin as not
   matched.
6. **Given** a connected account that already has an order waiting for payment in this shop,
   **When** the buyer checks out again with it, **Then** checkout is refused with a message that
   names the waiting order's reference; it works again once that order is paid or expired.

---

### User Story 2 - Unpaid orders expire and give their stock back (Priority: P1)

An order that is not paid within the payment window expires. Its items' stock goes back up by the
ordered quantities, so other buyers can buy them, and the order shows as **Expired** to the buyer
and the shop admin.

**Why this priority**: Since spec 003, every placed order takes stock for good; without expiry a
shop runs out of stock through abandoned orders.

**Independent Test**: Place an order, let the window pass without paying, and check that the order
is Expired and every item's stock is back to its earlier value, exactly once.

**Acceptance Scenarios**:

1. **Given** an order waiting for payment, **When** the payment window (30 minutes) passes with no
   matching payment, **Then** within 1 minute it is Expired and its stock is returned.
2. **Given** an order that expired, **When** the platform restarts or two copies of the shop check
   at the same time, **Then** its stock is returned only once.
3. **Given** an order that was paid just before the window ended, **When** the window passes,
   **Then** it stays Paid and its stock is not returned.
4. **Given** a payment that settles after its order expired, **When** the platform sees it,
   **Then** the order stays Expired, its stock is not taken again, and the transfer is listed to
   the shop admin as not matched.

---

### User Story 3 - The platform keeps every payment path open (Priority: P2)

Changing a shop or its settings never loses a payment or a shop's data. This story carries the
four follow-ups spec 004 deferred here (004 plan.md D-1, D-4, D-5, D-6):

- **Old wallet addresses stay until their orders are settled (D-1).** After an owner changes a
  shop's wallet address, the old wallet record is kept until every copy of the shop takes orders
  with the new address **and** every order that carries the old address is paid or expired. Only
  then is it removed.
- **"Ready" means every copy runs the current settings (D-5).** A shop is reported ready only when
  no copy with earlier settings (for example the old wallet address) is still serving.
- **The panel never shows an outdated "ready" (D-6).** Right after an owner changes a shop, the
  panel shows it as being prepared until the platform has picked up the change.
- **A shop can't be deleted outside ShopHub (D-4).** An attempt to delete a shop's resource
  directly, rather than through ShopHub, is refused, so its data (items, orders, admin password)
  stays; only a delete the owner asked for in ShopHub removes it.

**Why this priority**: Real payments make these gaps matter: an order placed during a rollout, or
an unpaid order on an old address, must still be payable and recognised.

**Independent Test**: Change a shop's wallet address while an order on the old address is unpaid
and the shop is rolling out; check the shop is not reported ready while an old copy serves, the
panel shows "being prepared" at once, the old wallet stays until that order is paid or expired,
and paying it marks it Paid. Separately, try to delete a shop's resource by hand and check it is
refused and the shop is untouched.

**Acceptance Scenarios**:

1. **Given** an unpaid order on the old address after a wallet change, **When** the buyer pays it
   to the old address, **Then** it becomes Paid; the old wallet record still exists until then.
2. **Given** a wallet change, **When** the last old-address order is paid or expired and every
   copy runs with the new address, **Then** the old wallet record is removed within 5 minutes.
3. **Given** a shop rolling out a change, **When** any copy with the old settings still serves,
   **Then** the shop is not reported ready, in the cluster or in the panel.
4. **Given** the owner has just saved a change, **When** the panel shows the shop before the
   platform picked it up, **Then** it shows "being prepared", not "ready".
5. **Given** a shop running normally, **When** someone deletes its resource directly instead of
   through ShopHub, **Then** the delete is refused with a message saying shops are deleted through
   ShopHub, and the shop keeps serving with the same items, orders and admin password.
6. **Given** an owner deletes a shop in ShopHub (004 FR-013), **When** the delete completes,
   **Then** everything is removed as before, within 2 minutes.

---

### User Story 4 - Shop admin sees payments (Priority: P3)

The shop admin's order list shows each order's payment status (waiting, Paid, Expired), and for a
paid order the time, the paying account and a link to the transaction on a public Sepolia
explorer.

**Why this priority**: Admins need to know what to ship; the data comes from Stories 1 and 2.

**Independent Test**: With one paid, one expired and one waiting order, the admin list shows the
three statuses and the paid order's transaction link.

**Acceptance Scenarios**:

1. **Given** paid, expired and waiting orders, **When** the admin lists orders, **Then** each shows
   its status, and the paid one its payment time, paying account and transaction link.

---

### Edge Cases

- **Two payments for one order** (double click, or paying again from another tab): the order is
  marked Paid once; the second transfer is recorded as not matched to any order, visible to the
  admin. The platform can't refund it (it holds no keys); the owner settles it outside the
  platform.
- **A second checkout from the same account**: refused while that account has an order waiting
  for payment in the shop (FR-005a), so one account never has two orders a transfer could fit.
  Another account, or the same account in another shop, may check out.
- **A payment seen and then undone by the chain** (a short reorganisation): an order counts as
  Paid only once its payment is final enough (6 blocks, FR-006), so it is never reversed
  after being shown as Paid.
- **The platform or the chain connection is down for a while**: payments that settled meanwhile
  are recognised after recovery, as long as their orders hadn't expired before the payment
  settled. Orders do not expire while the platform can't tell whether they were paid.
- **Buyer pays from a different account than the one they connected**: the transfer doesn't
  match; the order stays waiting (and expires), and the transfer is listed to the admin as not
  matched. The order page only proposes payment from the order's account.
- **Shop deleted in ShopHub while orders wait for payment**: the delete goes ahead (owner's
  choice); later transfers still reach the owner's own address, and nothing is recorded for them.
- **Wallet address changed twice quickly**: each old address is kept until its own orders are
  settled; none is removed early.
- **Item deleted or price changed after an order was placed**: the order keeps its own lines and
  total (spec 003); payment and expiry use those.

## Requirements *(mandatory)*

### Functional Requirements

**Paying (Requirement 2.4, User Story 1)**

- **FR-001**: Orders MUST be paid in mUSDT, the project's own test token on Ethereum Sepolia; ETH
  is used only for gas. One unit of a shop's price is one mUSDT. Mainnet and real-value tokens
  MUST NOT be accepted (Principle IV).
- **FR-002**: The order page (reached by the order reference) MUST show the total in mUSDT, the
  shop's wallet address for this order, the time left to pay and the status, and MUST offer
  "Pay with MetaMask", which proposes a transfer of exactly the total to that address on Sepolia.
- **FR-002a**: Checkout MUST ask the buyer to connect MetaMask and MUST record the connected
  account as the order's paying account.
- **FR-003**: Each order MUST record the wallet address it is to be paid to when it is placed
  (spec 003 already stores it); later wallet changes MUST NOT change it.
- **FR-004**: The platform MUST recognise a payment from the chain itself, without relying on the
  buyer's browser staying open: a settled transfer of mUSDT to an order's address that matches the
  order (FR-005) marks it **Paid**, with the transaction, paying account, amount and time.
- **FR-005**: A transfer MUST match at most one order, and an order at most one transfer. A
  transfer of another token, to another address, of an amount other than the exact total, or
  settled before the order was placed MUST NOT mark an order Paid. Matching rule: a transfer pays
  an order when it is of mUSDT, from the order's paying account, to the order's address, for
  exactly the order's total, and settled after the order was placed while the order was waiting
  for payment.
- **FR-005a**: An account MUST NOT have more than one order waiting for payment in the same shop.
  Checkout with an account that already has one MUST be refused, naming that order's reference,
  until it is paid or expired. This holds with several copies of the shop and concurrent
  checkouts.
- **FR-006**: A payment MUST count only once 6 more blocks have followed the one that holds it (a
  platform setting, default 6). The platform MUST check the chain for new blocks at least every
  12 seconds. The order page MUST show "payment sent, confirming" in between when the
  buyer's browser reported the transaction.
- **FR-007**: Transfers to a shop's address that match no order (wrong amount, a second payment,
  after expiry where FR-010 doesn't accept them) MUST be recorded and listed to the shop admin.
- **FR-008**: After an outage of the platform or its chain connection, payments that settled
  meanwhile MUST still be recognised, from where the platform left off, without counting any
  transfer twice.

**Expiry (User Story 2)**

- **FR-009**: An order waiting for payment MUST expire 30 minutes after it was placed (a platform
  setting). Expiring MUST set it to **Expired** and return each line's quantity to its item's stock
  in one step, exactly once, even with several copies of the shop or a restart. An item deleted
  since then gets nothing back.
- **FR-010**: A payment that settles after its order expired MUST NOT revive the order or take
  stock again; it MUST be recorded as not matched (FR-007) for the admin, who settles it outside
  the platform.
- **FR-011**: An order MUST NOT expire while the platform is behind on reading the chain past the
  order's deadline; it expires only once the platform has seen every block up to that deadline
  without a matching payment.

**Keeping payment paths and data (User Story 3; 004 D-1, D-4, D-5, D-6)**

- **FR-012** (D-1): After a wallet change, the old wallet record MUST be kept until every copy of
  the shop runs with the new address **and** no order carrying the old address is still waiting
  for payment; then it MUST be removed within 5 minutes. This replaces 004 FR-009's "the old
  Wallet's deletion waits until no Shop uses it".
- **FR-013** (D-5): A shop MUST be reported ready only when every running copy has the shop's
  current settings; a copy with earlier settings still serving keeps it not ready.
- **FR-014** (D-6): The panel MUST show a shop as being prepared while the platform has not yet
  picked up its latest change, whatever an earlier status said.
- **FR-015** (D-4): A deletion of a shop's resource that ShopHub didn't ask for MUST be refused,
  so the resource and its data stay; only a shop ShopHub has marked for deletion (004 FR-013) can
  be deleted. A delete the owner asked for in ShopHub MUST still remove everything within 2
  minutes (004 SC-003).

**Admin view (User Story 4)**

- **FR-016**: The shop admin's order list MUST show each order's status (waiting for payment,
  Paid, Expired) and for paid orders the payment time, paying account and a link to the
  transaction on a public Sepolia explorer; and it MUST list unmatched transfers (FR-007).

**Platform and safety**

- **FR-017**: The mUSDT token MUST be deployed once on Sepolia by the maintainer, by hand from
  their own MetaMask account (Remix), with no deploying key stored by the project; tests deploy
  the same contract on the local chain. Its source lives in a project repository with its tests. It MUST let any account mint itself up to 1,000 mUSDT per call, since it
  has no value; the order page MUST link to that as "Get test mUSDT". The platform runs no faucet
  and holds no funded account. Its address is a public platform setting recorded in kube-state and
  given to every shop; it is not a Secret.
- **FR-018**: The platform MUST NOT hold, ask for or store any private key of buyers or owners.
  Reading the chain MUST need no private key. The platform reads Sepolia through a provider
  endpoint whose URL contains an API key; that URL is a Secret made by hand once per cluster (like
  the Discord token) and MUST never appear in a repository, log, resource status or API response
  (004 SC-008 extended).
- **FR-020**: Catching up after an outage (FR-008) MUST work within a free provider plan's limits
  on how many blocks one query may cover. The part of the platform that reads the chain MUST have
  an explicit allowance to reach the provider, so network restrictions (today incoming-only, 004)
  never cut off payment detection.
- **FR-019**: Every automated test touching the token or the chain MUST run against a local chain;
  CI MUST NOT use Sepolia. Sepolia is used only for the live end-to-end check (Principle IV).

### Key Entities *(include if feature involves data)*

- **Order** (from spec 003): reference, lines, total, the wallet address it is to be paid to, the
  paying account connected at checkout (at most one waiting order per account and shop), the time
  placed, and now a payment status — waiting for payment → Paid or Expired — with a deadline
  and, once paid, the payment that settled it.
- **Payment**: a settled mUSDT transfer seen on the chain to a shop's address: transaction, block,
  paying account, receiving address, amount, time, and the order it matched, or none.
- **mUSDT**: the project's test token on Sepolia (and on the local test chain); free to obtain;
  its address is a platform setting.
- **Wallet** (from specs 002/004): the shop's receiving address. An old one now outlives a wallet
  change until its orders are settled (FR-012).
- **Shop** (from specs 002/004): now reported ready only when every copy runs current settings
  (FR-013), and can be deleted only after ShopHub has marked it for deletion (FR-015).

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A buyer with mUSDT and Sepolia ETH can pay an order from its page in under 2 minutes
  of their own time, and sees it as Paid within 2 minutes of confirming in MetaMask.
- **SC-002**: 100% of exact payments to an order's address mark that order Paid exactly once; 0
  transfers of the wrong token, amount, address or paying account, or duplicates, mark an order
  Paid; 0 accounts ever hold two waiting orders in one shop. Checked by
  automated tests on a local chain for each case.
- **SC-003**: Unpaid orders are Expired within 1 minute after their deadline and their stock is
  back exactly once, also with 3 copies of the shop and across a restart.
- **SC-004**: After the platform or its chain connection has been down for up to 1 hour,
  payments made meanwhile are recognised within 5 minutes of recovery, and no order paid before
  its deadline is expired.
- **SC-005**: During and after a wallet change, 0 unpaid orders are left with an address whose
  wallet record is gone, and every old wallet record is gone within 5 minutes after its last
  order is settled.
- **SC-006**: After a wallet change, 0 orders are placed with the old address once the shop is
  reported ready; the panel shows "being prepared" within 5 seconds of the save until then.
- **SC-007**: 100% of attempts to delete a shop's resource outside ShopHub are refused, and the
  shop keeps all of its items, orders and its admin password.
- **SC-008**: 0 private keys and 0 chain-access credentials in any repository, log, resource
  status or API response; 0 CI runs contact Sepolia.

## Assumptions

- **Prices in mUSDT**: a shop's prices (two decimals, spec 003) are amounts of mUSDT; no
  conversion from any other currency.
- **Payment window**: 30 minutes, a platform setting; the same for every shop.
- **Final enough**: 6 blocks after the payment's block (about 72 s on Sepolia), configurable;
  with checks at least every 12 s this keeps SC-001's 2 minutes.
- **Getting test tokens**: buyers get Sepolia ETH from a public faucet and mint mUSDT themselves
  from the token (FR-017). Testers use their own MetaMask accounts.
- **No refunds**: the platform holds no keys, so double or late payments are only recorded and
  shown to the admin; the owner settles them outside the platform.
- **Buyers have no accounts**: as in spec 003, an order is reached by its reference. Checkout
  now needs a connected MetaMask account (FR-002a), but that is not a sign-in.
- **Out of scope**: notifications of paid orders on the shop's notification channel, refunds,
  partial payments, other tokens or chains, Web3 sign-in (optional in the requirement), Wallet
  logging (004 D-3, spec 006) and observability (spec 006).

## Dependencies

- Spec 003: orders, stock, order references; the shop app records each order's wallet address.
- Spec 004: Wallet per shop with the owner's address; ShopHub as the only writer of Shops and
  Wallets; deferred items D-1, D-4, D-5, D-6 (plan.md).
- Spec 002: the Wallet, Shop and DiscordChannel resources and the shop-operator; FR-012, FR-013
  and FR-015 change how it keeps Wallets, reports readiness and handles a Shop deleted outside
  ShopHub, and may need a new shop-operator release.
- A Sepolia provider account (free plan) and its endpoint URL with API key, kept as a hand-made
  Secret.
- Constitution Principle IV (Sepolia, MetaMask, mUSDT, local chain for tests) and Principle V
  (user resources only through ShopHub; derived resources through the operator).
