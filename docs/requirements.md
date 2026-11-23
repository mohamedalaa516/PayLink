# PayLink — Requirements Document

## 1. Problem Summary
PayLink is a digital wallet and bill-payment backend for Egypt. Customers use it through a mobile app to pay utility bills (electricity, water, mobile, and internet), and local shop agents can also process payments through terminals.
PayLink makes money by charging a small service fee on each payment and pays agents a commission for their transactions.

- The hard part of this system is making sure no money is ever lost or miscounted when things go wrong. Before paying any bill, the backend has to check that the user has enough balance and hold that amount until the biller confirms the payment.
- It also has to prevent double charges if a request or callback is sent twice, and handle two payments coming from the same wallet at the exact same time so the balance never goes negative.
- On top of that, the system needs to deal with biller timeouts, rejections, failed notifications, and nightly settlement checks to catch any mismatches.
- Finally, it should be built so we can easily connect new billers with different APIs without changing the rest of the codebase, and handle heavy traffic on busy days of the month.

## 2. Stakeholders
- **Customers (Basic & Verified):** End users who top up their wallets, inquire about bills, pay utilities, view transaction history, and receive payment notifications.
- **Agents:** Small shop owners with PayLink terminals who process bill payments on behalf of customers and earn a commission on completed payments.
- **Billers (ElectroCo, AquaCo, TopUpNet, NetCo):** External utility and telecom companies that provide bill inquiries, process payments, and send daily settlement files.
- **Support Employees:** Internal staff who investigate payment histories by reference, review reconciliation discrepancies, approve refund requests, and block/unblock accounts.
- **System Administrators (Admins):** Internal staff who manage the biller catalog (add, edit, disable services), configure fees/limits/promotions without redeployment, and monitor daily financial totals.
- **PayLink Management & Regulators:** Business owners and financial regulators who require strict financial accuracy, full audit logs, and compliance.

## 3. Ambiguities and Decisions

1. **Daily Limit Reset Window and Timezone**
   - **Ambiguity:** The brief specifies daily limits (5,000 EGP basic / 30,000 EGP verified) but does not define when the day resets.
   - **Decision:** Reset daily limits at `00:00` Cairo local time (`Africa/Cairo`) to match the customer's actual calendar day in Egypt.

2. **Scope of Payment Limits (Bill Amount vs. Total with Fee)**
   - **Ambiguity:** It is unclear whether the single-payment limits (5–10,000 EGP) and daily limits apply to the bill amount alone or include PayLink's service fee.
   - **Decision:** Apply single-payment and daily limits to the bill principal amount (excluding service fees), which is standard for bill-payment platforms and easier for users to understand.

3. **Wallet Source and Limits for Agent-Initiated Payments**
   - **Ambiguity:** The brief does not state whose wallet is debited when an agent pays a bill on behalf of a customer.
   - **Decision:** Debit the agent's own wallet (since the customer pays cash at the shop, similar to Fawry) and apply a separate configurable agent daily limit.

4. **Velocity Rule Counting Criteria**
   - **Ambiguity:** It is unclear whether failed, rejected, or duplicate idempotent requests count toward the "5 payments in 60 seconds" velocity rule.
   - **Decision:** Count all unique initiated payment attempts, excluding exact idempotency-key retries. This blocks rapid spam without punishing users on slow networks.

5. **Idempotency Key Scope, Expiration, and Payload Mismatch**
   - **Ambiguity:** The brief does not specify how long idempotency keys live or what happens if a key is reused with different parameters.
   - **Decision:** Scope keys per `user_id` with a 24-hour TTL and store a hash of the request payload. If the same key is reused with different data, return `409 Conflict`.

6. **Post-Completion Refunds vs. Immutable Final States**
   - **Ambiguity:** Section 2.5 defines `Completed` as a final state, while FR-27 allows customers to request a refund for a completed payment with support approval.
   - **Decision:** Keep the original payment permanently in `Completed` so the state machine stays strict. Model FR-27 as a separate `RefundRequest` entity that creates a compensating ledger credit upon support approval (and reverses agent commission if applicable).

7. **Wallet Top-Up Verification and Constraints**
   - **Ambiguity:** Top-up min/max limits, fees, and sync/async behavior are not specified.
   - **Decision:** Implement simulated card and agent-cash top-ups as synchronous and fee-free, protected by idempotency keys, with configurable min/max bounds (e.g., 10 EGP – 30,000 EGP).

8. **Bill Inquiry Expiration and Enforcement**
   - **Ambiguity:** The brief does not state whether a bill inquiry is mandatory before paying or how long an inquiry result stays valid.
   - **Decision:** Keep bill inquiry stateless and optional at the API level; the payment endpoint validates limits, recalculates the current fee at execution time, and relies on the biller adapter to validate the amount.

9. **Reconciliation Mismatch Resolution Workflow**
   - **Ambiguity:** It is unclear if support resolves nightly settlement mismatches inside the automated state machine or through a separate workflow.
   - **Decision:** Flag mismatches in the daily reconciliation report for support review (matching Hard Case #7) and allow support to log an auditable manual ledger adjustment without breaking the payment state machine.

10. **Service Fee Rounding Precision**
    - **Ambiguity:** Percentage fees (1.5% on mobile top-up) can produce fractions of a piaster, and no rounding mode is specified.
    - **Decision:** Round all calculated fees and commissions to 2 decimal places (piasters) using standard Half-Up rounding (`ROUND_HALF_UP`) with exact decimal types.

## 4. Functional Requirements — User Stories (FR-1 to FR-15)

### FR-1: Customer Registration (`Must`)

**User Story:** As a new customer, I want to register an account with a default basic verification level, so that I can own a digital wallet and start paying utility bills.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Successful Registration):**
  - **Given** a new user provides valid registration details and does not already have an account,
  - **When** they submit the registration request,
  - **Then** the system creates the customer account with a `basic` verification level, enforces a daily payment limit of `5,000.00 EGP`, and initializes a wallet with `0.00 EGP` available balance and `0.00 EGP` held balance.

- **Scenario 2 (Failure/Edge Case — Duplicate Registration):**
  - **Given** a customer account already exists for the provided unique identifier (e.g., phone number or email),
  - **When** a registration request is submitted with the same identifier,
  - **Then** the system rejects the request with a conflict error and does not create a duplicate account or wallet.

---

### FR-2: Account Verification Upgrade (`Should`)

**User Story:** As a customer with a basic account, I want my account to be upgraded to verified, so that my daily payment limit is raised to `30,000.00 EGP`.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Upgrade to Verified):**
  - **Given** an active customer account has a `basic` verification level with a `5,000.00 EGP` daily limit,
  - **When** an authorized verification upgrade is completed for that account,
  - **Then** the customer's verification level changes to `verified`, their daily payment limit increases to `30,000.00 EGP`, and the action is recorded in the audit log.

- **Scenario 2 (Failure/Edge Case — Already Verified or Blocked Account):**
  - **Given** a customer account is already `verified` or is currently blocked,
  - **When** a verification upgrade is requested,
  - **Then** the system rejects the request with a clear validation error and leaves the account state unchanged.

---

### FR-3: View Wallet Balance and Transaction History (`Must`)

**User Story:** As a customer, I want to view my wallet's available balance, held balance, and transaction history, so that I can track my funds and verify pending or completed payments.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Balance and History Retrieval):**
  - **Given** a customer has completed a top-up of `1,000.00 EGP` and currently has a pending payment holding `352.50 EGP`,
  - **When** the customer requests their wallet summary and transaction history,
  - **Then** the system returns an available balance of `647.50 EGP`, a held balance of `352.50 EGP` (computed using exact decimal values), and a chronological list of their wallet transactions.

- **Scenario 2 (Failure/Edge Case — Empty History on New Wallet):**
  - **Given** a newly registered customer has made no top-ups or payments,
  - **When** they request their wallet balance and transaction history,
  - **Then** the system returns `0.00 EGP` available, `0.00 EGP` held, and an empty transaction list without error.

---

### FR-4: Wallet Top-Up (`Must`)

**User Story:** As a customer, I want to top up my wallet via a simulated card or agent cash, so that I have sufficient available funds to pay my utility bills.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Successful Top-Up):**
  - **Given** an active customer has an available balance of 100.00 EGP,
  - **When** they submit a valid synchronous top-up of 500.00 EGP (within configurable min/max bounds of 10–30,000 EGP) with a unique idempotency key via simulated card or agent cash,
  - **Then** a fee-free credit ledger entry of 500.00 EGP is recorded, the available balance becomes 600.00 EGP, and an audit record is created.

- **Scenario 2 (Failure/Edge Case — Invalid Amount, Out of Bounds, or Blocked Account):**
  - **Given** a customer submits a top-up outside the configured min/max bounds (e.g., -50.00 EGP or 50,000.00 EGP) or their account is blocked,
  - **When** the top-up request is processed,
  - **Then** the system rejects the request with a validation error and creates no ledger entry.

---

### FR-5: Ledger-Derived Balances (`Must`)

**User Story:** As a system architect, I want every wallet balance change to be recorded as an immutable ledger entry and all balances to be computed from the ledger, so that money movements are auditable and never silently overwritten.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Balance Computed from Ledger Entries):**
  - **Given** a wallet has ledger entries for a `+500.00 EGP` top-up, a `-202.50 EGP` hold (moving funds from available to held), and a final charge upon completion,
  - **When** the system calculates the wallet's available and held balances,
  - **Then** the returned balances equal the exact decimal sum of the corresponding ledger entries, with no direct overwrites to a balance column.

- **Scenario 2 (Failure/Edge Case — Concurrent Spend Attempt):**
  - **Given** a wallet has an available balance of `400.00 EGP`,
  - **When** two concurrent payment requests of `352.50 EGP` arrive at the exact same moment,
  - **Then** the ledger commits the hold for only one payment, the second payment is `Rejected`, and the computed available balance never goes negative.

---

### FR-6: Biller and Service Administration (`Must`)

**User Story:** As an admin, I want to add, edit, and disable billers and their services, so that PayLink can onboard new utility providers and suspend unavailable services without redeploying.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Add and Edit Biller):**
  - **Given** an authenticated admin submits a new biller configuration (name, service type, required input fields, min/max amounts, active status),
  - **When** the admin saves or updates the biller,
  - **Then** the biller is persisted in the catalog, becomes available for customer inquiries/payments, and the admin action is recorded in the audit log.

- **Scenario 2 (Failure/Edge Case — Disabled Biller Rejection):**
  - **Given** an admin has disabled an existing biller (e.g., AquaCo),
  - **When** a customer attempts to run a bill inquiry or initiate a payment for that disabled biller,
  - **Then** the system rejects the request with a clear service-unavailable error and does not hold any funds.

---

### FR-7: Browse Biller Catalog and Required Inputs (`Must`)

**User Story:** As a customer, I want to view the list of active billers and the input fields each requires, so that I know which details (such as meter number or phone number) to provide before paying.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — List Active Billers):**
  - **Given** ElectroCo (requiring `meter_number`), AquaCo, and TopUpNet (requiring `phone_number`) are active in the catalog,
  - **When** a customer requests the biller catalog,
  - **Then** the system returns all active billers along with their service types, required input fields, and min/max payment bounds.

- **Scenario 2 (Failure/Edge Case — Disabled Billers Excluded):**
  - **Given** a biller has been marked as disabled by an admin,
  - **When** a customer requests the biller catalog,
  - **Then** the disabled biller is excluded from the active list shown to the customer.

---

### FR-8: Bill Inquiry with Fee Preview (`Must`)

**User Story:** As a customer, I want to run a bill inquiry and see the bill amount, service fee, and total cost before paying, so that I can confirm what I owe prior to authorizing the charge.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Successful Inquiry):**
  - **Given** ElectroCo is active and has an unpaid bill of `350.00 EGP` for meter number `123456`,
  - **When** the customer runs a bill inquiry for meter `123456`,
  - **Then** PayLink queries ElectroCo, calculates the `2.50 EGP` electricity service fee, and displays `350.00 EGP` (amount), `2.50 EGP` (fee), and `352.50 EGP` (total).

- **Scenario 2 (Failure/Edge Case — Invalid Reference or Unreachable Biller):**
  - **Given** the customer enters a non-existent meter number or the biller returns an error/timeout,
  - **When** the inquiry is submitted,
  - **Then** the system returns a clear error message explaining the failure and does not initiate a payment or hold funds.

---

### FR-9: Pay a Bill from Wallet (`Must`)

**User Story:** As a customer, I want to pay a utility bill directly from my wallet, so that my bill is settled quickly without visiting a physical office.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Successful Immediate Payment):**
  - **Given** a customer has `500.00 EGP` available, passes all fraud and limit checks, and submits a `350.00 EGP` ElectroCo payment (total `352.50 EGP` with fee),
  - **When** ElectroCo confirms the payment within the 10-second timeout,
  - **Then** the payment reaches `Completed`, the `352.50 EGP` hold becomes a final charge, PayLink credits `2.50 EGP` to fee income, and the customer's available balance is `147.50 EGP`.

- **Scenario 2 (Failure/Edge Case — Insufficient Funds):**
  - **Given** a customer has `100.00 EGP` available and attempts to pay a `350.00 EGP` bill (total `352.50 EGP`),
  - **When** the payment is submitted,
  - **Then** the payment transitions from `Initiated` to `Rejected`, no funds are held, and the biller is never called.

---

### FR-10: Idempotent Payment Processing (`Must`)

**User Story:** As a customer on an unstable mobile network, I want repeated payment requests with the same idempotency key to return the original result, so that I am never charged twice for the same bill.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Duplicate Request Returns Original Result):**
  - **Given** a payment request with idempotency key `idem-001` has already been processed and returned a response,
  - **When** the client sends the same request again with idempotency key `idem-001` and identical payload within the 24-hour TTL,
  - **Then** the system does not create a second payment, hold, or biller call, and returns the exact original result.

- **Scenario 2 (Edge Case — Concurrent Duplicate Request):**
  - **Given** a payment request with idempotency key `idem-002` is currently in flight,
  - **When** an identical request with `idem-002` arrives before the first finishes,
  - **Then** the system ensures only one payment and one wallet hold are executed, preventing any double charge.

- **Scenario 3 (Failure Case — Idempotency Key Reused with Different Payload):**
  - **Given** a customer already used idempotency key `idem-001` for a 350.00 EGP ElectroCo payment,
  - **When** the same customer sends a new request with `idem-001` but a different amount or biller reference,
  - **Then** the system rejects the request with `409 Conflict` and does not execute a new payment.

---

### FR-11: Strict Payment State Machine Enforcement (`Must`)

**User Story:** As a backend engineer, I want every payment to follow only the defined state transitions in Section 2.5, so that payments cannot enter invalid financial states or be double-refunded.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Valid State Progression):**
  - **Given** a payment is created in `Initiated`,
  - **When** fraud/balance checks pass, the biller request is sent, and the biller confirms,
  - **Then** the payment transitions strictly through `Initiated -> FundsHeld -> SentToBiller -> Completed`, and each transition is logged.

- **Scenario 2 (Failure/Edge Case — Illegal State Transition Blocked):**
  - **Given** a payment is in a final state (`Completed`, `Rejected`, or `Refunded`) or in `Unknown`,
  - **When** an operation attempts an invalid transition (e.g., `Unknown -> Refunded` directly, or `Completed -> Failed`),
  - **Then** the state machine rejects the transition with an invalid-state error and leaves the payment state and ledger unchanged.

---

### FR-12: Funds Hold Before Biller Call and Release on Failure (`Must`)

**User Story:** As a customer, I want my funds to be held before the biller is contacted and automatically released if the biller refuses the payment, so that my money is protected during processing and promptly returned on failure.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Hold and Release on Biller Refusal):**
  - **Given** a customer has `500.00 EGP` available and pays an AquaCo bill of `200.00 EGP` (total `202.50 EGP`),
  - **When** the system moves `202.50 EGP` to held (`FundsHeld`), sends the request (`SentToBiller`), and AquaCo refuses with an error code,
  - **Then** the payment transitions to `Failed` and then `Refunded`, the `202.50 EGP` hold is released back to the available balance (`500.00 EGP`), and a refund ledger entry is recorded.

- **Scenario 2 (Failure/Edge Case — Hold Preserved on Unknown Outcome):**
  - **Given** funds of `202.50 EGP` are held and the payment transitions to `Unknown` after a biller timeout,
  - **When** the timeout occurs,
  - **Then** the `202.50 EGP` remains in the held balance and is **not** released until the biller refuses or reconciliation confirms the biller did not receive the payment.

---

### FR-13: Timeout Handling and Unknown Payment Retries (`Must`)

**User Story:** As a payment operations engineer, I want payments that exceed the biller timeout to become `Unknown` and be re-queried on a fixed retry schedule, so that ambiguous transactions are resolved without premature refunds or double payments.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Timeout Resolved by Retry):**
  - **Given** ElectroCo takes `12 seconds` to respond (exceeding the `10-second` live biller timeout),
  - **When** the timeout triggers,
  - **Then** the payment moves to `Unknown`, the money stays held, the customer is informed it is being processed, and the system schedules status re-queries after `1`, `5`, and `15` minutes; if a re-query returns confirmation, the payment transitions to `Completed`.

- **Scenario 2 (Failure/Edge Case — Exhausted Retries Wait for Reconciliation):**
  - **Given** an `Unknown` payment has been re-queried `3` times (at `1`, `5`, and `15` minutes) without a definitive confirmation or refusal,
  - **When** the third retry still yields no final status,
  - **Then** the system stops automatic retries, keeps the payment in `Unknown` with funds held, and leaves it to be resolved by nightly reconciliation.

---

### FR-14: Asynchronous Callbacks from TopUpNet with Deduplication (`Must`)

**User Story:** As a customer topping up my mobile line via TopUpNet, I want PayLink to process asynchronous completion callbacks and ignore duplicate notifications, so that my payment finalizes accurately once.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Async Acceptance and Success Callback):**
  - **Given** a TopUpNet payment receives an initial `"accepted"` response and remains in progress with funds held,
  - **When** TopUpNet invokes PayLink's callback URL minutes later with status `"success"`,
  - **Then** the payment transitions to `Completed`, the held funds are finalized as a charge, and the customer is notified.

- **Scenario 2 (Edge Case — Duplicate Success Callback):**
  - **Given** TopUpNet has already sent a `"success"` callback that moved the payment to `Completed`,
  - **When** TopUpNet sends the same `"success"` callback a second time,
  - **Then** the system ignores the duplicate callback without modifying the balance or state, logs the duplicate event, and returns a safe acknowledgment.

- **Scenario 3 (Failure Case — Failure Callback):**
  - **Given** a TopUpNet payment is awaiting an asynchronous callback with funds held,
  - **When** TopUpNet invokes PayLink's callback URL with status `"failed"`,
  - **Then** the payment transitions to `Failed` and then `Refunded`, releasing the held funds back to the customer's available balance.

---

### FR-15: Multi-Protocol Biller Integrations (`Must`)

**User Story:** As a software architect, I want PayLink to integrate with ElectroCo (REST/JSON), AquaCo (SOAP/XML), and TopUpNet (async REST/JSON) behind isolated biller adapters, so that protocol differences are hidden from core payment processing and new billers can be added cleanly.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Uniform Core Flow Across Protocols):**
  - **Given** ElectroCo (sync REST/JSON), AquaCo (SOAP-style XML over HTTP), and TopUpNet (async REST/JSON with callback) are configured,
  - **When** payments are submitted to each of the three billers through the same PayLink payment API,
  - **Then** each biller's adapter translates the request/response into its required protocol and format, and the core Payment Processing module handles all three without protocol-specific branching.

- **Scenario 2 (Failure/Edge Case — Biller-Specific Rejection & Isolation):**
  - **Given** AquaCo rejects a payment exceeding the bill amount with a SOAP XML error code, or ElectroCo experiences an outage,
  - **When** the error occurs,
  - **Then** the AquaCo adapter maps the XML error into a standardized refusal (triggering `Failed` -> `Refunded`), and an outage in one biller does not prevent payments to the other active billers.