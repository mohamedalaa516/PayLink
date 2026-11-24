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

11. **Scope of Blocked Account Restrictions**
    - **Ambiguity:** Section 2.2 Step 4 checks for blocked accounts during bill payments, but does not state whether blocked accounts can still top up their wallets or upgrade verification levels.
    - **Decision:** Freeze blocked accounts across all sensitive operations—blocked accounts cannot initiate payments, top up their wallets, or undergo verification upgrades until unblocked by support.
    
## 4. Functional Requirements — User Stories (FR-1 to FR-28)

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

---

### FR-16: Service Fee Calculation and Promotion Rules (`Must`)

**User Story:** As a customer, I want service fees and active promotions to be calculated accurately according to the biller type and bill amount, so that I am charged the exact expected fee at payment execution time.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Fixed Electricity and Water Fees):**
  - **Given** no active promotion is configured for ElectroCo or AquaCo,
  - **When** a customer pays a `350.00 EGP` ElectroCo (electricity) bill and a `150.00 EGP` AquaCo (water) bill,
  - **Then** PayLink calculates a fixed service fee of `2.50 EGP` for each payment.

- **Scenario 2 (Happy Path — Percentage Mobile Fee with `ROUND_HALF_UP`):**
  - **Given** no active promotion is configured for TopUpNet,
  - **When** a customer pays a `103.00 EGP` TopUpNet (mobile) top-up (`1.5%` = `1.545 EGP`),
  - **Then** PayLink rounds to 2 decimal places using `ROUND_HALF_UP` and charges a service fee of `1.55 EGP` (not `1.54 EGP`).

- **Scenario 3 (Edge Case — Minimum Mobile Fee Boundary):**
  - **Given** no active promotion is configured for TopUpNet,
  - **When** a customer pays a `50.00 EGP` TopUpNet mobile top-up (`1.5%` = `0.75 EGP`),
  - **Then** PayLink enforces the `1.00 EGP` minimum fee and charges `51.00 EGP` total.

- **Scenario 4 (Edge Case — Tiered Internet Fee Boundary Values):**
  - **Given** no active promotion is configured for NetCo,
  - **When** a customer pays a `200.00 EGP` NetCo (internet) bill and a `200.01 EGP` NetCo bill,
  - **Then** PayLink charges a `2.00 EGP` fee for the `200.00 EGP` bill and a `4.00 EGP` fee for the `200.01 EGP` bill.

- **Scenario 5 (Edge Case — Active Promotion Window):**
  - **Given** an admin has configured a promotion setting the fee to `0.00 EGP` for ElectroCo between `2026-12-01 00:00` and `2026-12-07 23:59` (`Africa/Cairo`),
  - **When** a customer pays a `350.00 EGP` ElectroCo bill during the active promotion window,
  - **Then** the recalculated service fee at payment time is `0.00 EGP`, the customer is charged `350.00 EGP` total, and payments to other billers during the same period still incur their standard fees.

---

### FR-17: Agent-Assisted Bill Payment and Commission (`Should`)

**User Story:** As an agent, I want to pay a utility bill on behalf of a customer from my own wallet and earn a commission when the payment completes, so that I am compensated for processing terminal payments.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Agent Payment Completed and Commission Credited):**
  - **Given** an active agent has `2,000.00 EGP` available in their own wallet (within their configured agent daily limit) and submits a `350.00 EGP` ElectroCo payment (`2.50 EGP` fee, `352.50 EGP` total) on behalf of a customer,
  - **When** ElectroCo confirms the payment and it transitions to `Completed`,
  - **Then** `352.50 EGP` is debited from the agent's wallet, `2.50 EGP` is credited to PayLink fee income, a commission ledger entry of `1.00 EGP` (`40%` of `2.50 EGP`, rounded `ROUND_HALF_UP`) is credited to the agent's wallet, and the agent's remaining available balance is `1,648.50 EGP`.

- **Scenario 2 (Failure/Edge Case — Biller Refusal Yields Zero Commission):**
  - **Given** an active agent has `2,000.00 EGP` available, initiates a `200.00 EGP` AquaCo payment (`2.50 EGP` fee, `202.50 EGP` total), and `202.50 EGP` is held in the agent's wallet,
  - **When** AquaCo refuses the payment (`SentToBiller -> Failed -> Refunded`),
  - **Then** the `202.50 EGP` hold is released back to the agent's available balance (`2,000.00 EGP` available, `0.00 EGP` held) and `0.00 EGP` commission is credited to the agent.

---

### FR-18: Single-Payment and Daily Limit Enforcement (`Must`)

**User Story:** As a system operator, I want payments exceeding single-payment bounds or account daily limits to be rejected immediately with a clear reason, so that PayLink enforces transaction limits accurately.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Daily Limit Checked on Bill Principal Excluding Fee):**
  - **Given** a `basic` customer (`5,000.00 EGP` daily limit) has already paid `4,650.00 EGP` in bill principal today (`Africa/Cairo`) and submits a `350.00 EGP` ElectroCo payment (`2.50 EGP` fee, `352.50 EGP` total),
  - **When** limit validation executes,
  - **Then** PayLink evaluates the `350.00 EGP` bill principal excluding the `2.50 EGP` fee (`4,650.00 + 350.00 = 5,000.00 EGP`) and allows the payment to proceed.

- **Scenario 2 (Happy Path — Daily Limit Resets at `00:00 Africa/Cairo`):**
  - **Given** a `basic` customer has used `5,000.00 EGP` of their daily limit by `23:59` `Africa/Cairo` on `2026-11-23`,
  - **When** the customer initiates a `350.00 EGP` ElectroCo payment at `00:01` `Africa/Cairo` on `2026-11-24`,
  - **Then** PayLink evaluates their daily usage for the new Cairo calendar day as `0.00 EGP` and allows the payment.

- **Scenario 3 (Failure/Edge Case — Below Single-Payment Minimum):**
  - **Given** an active customer with `1,000.00 EGP` available submits a payment with a bill principal of `4.99 EGP` (below the `5.00 EGP` minimum),
  - **When** limit validation executes,
  - **Then** the payment transitions from `Initiated` to `Rejected` with a clear below-minimum-amount reason, zero funds are held, and the biller is not called.

- **Scenario 4 (Failure/Edge Case — Above Single-Payment Maximum):**
  - **Given** an active `verified` customer with `20,000.00 EGP` available submits a payment with a bill principal of `10,000.01 EGP` (above the `10,000.00 EGP` maximum),
  - **When** limit validation executes,
  - **Then** the payment transitions from `Initiated` to `Rejected` with a clear above-maximum-amount reason, zero funds are held, and the biller is not called.

- **Scenario 5 (Failure/Edge Case — Basic Customer Daily Limit Exceeded):**
  - **Given** a `basic` customer has used `4,800.00 EGP` of their `5,000.00 EGP` daily limit today (`Africa/Cairo`),
  - **When** the customer submits a `300.00 EGP` bill payment,
  - **Then** the payment transitions from `Initiated` to `Rejected` with a clear daily-limit-exceeded reason and no funds are held.

- **Scenario 6 (Failure/Edge Case — Verified Customer Daily Limit Exceeded):**
  - **Given** a `verified` customer has used `29,800.00 EGP` of their `30,000.00 EGP` daily limit today (`Africa/Cairo`),
  - **When** the customer submits a `300.00 EGP` bill payment,
  - **Then** the payment transitions from `Initiated` to `Rejected` with a clear daily-limit-exceeded reason and no funds are held.

- **Scenario 7 (Failure/Edge Case — Agent Daily Limit Exceeded):**
  - **Given** an active agent has `200.00 EGP` remaining under their separate admin-configured agent daily limit today (`Africa/Cairo`),
  - **When** the agent submits a `300.00 EGP` bill payment on behalf of a customer,
  - **Then** the payment transitions from `Initiated` to `Rejected` with a clear agent-daily-limit-exceeded reason and no funds are held in the agent's wallet.

---

### FR-19: Velocity Fraud Rule Enforcement (`Should`)

**User Story:** As a fraud prevention engineer, I want an account that initiates more than 5 unique payment attempts within 60 seconds to be automatically blocked for 10 minutes, so that rapid automated abuse is stopped.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — 6th Unique Payment Attempt Blocks Account for 10 Minutes):**
  - **Given** an active customer has initiated `5` unique payment attempts (with distinct idempotency keys) within the past `60 seconds`,
  - **When** the customer submits a `6th` unique payment attempt within that same `60-second` window,
  - **Then** the `6th` payment transitions from `Initiated` to `Rejected`, the account is automatically blocked for `10 minutes`, an audit record is created, and any new payment attempts during those `10 minutes` are rejected.

- **Scenario 2 (Edge Case — Duplicate Idempotency Retries Excluded from Velocity Count):**
  - **Given** a customer has initiated `5` unique payment attempts within `60 seconds` and resends the `5th` request `3` times with the exact same idempotency key and payload due to network lag,
  - **When** the velocity check evaluates the account,
  - **Then** the exact idempotency-key retries are excluded from the velocity counter, the original response is returned, and the account remains active.

---

### FR-20: Search Payment History by Reference (`Must`)

**User Story:** As a support employee, I want to search for a payment by its unique reference and inspect its complete history, so that I can investigate customer issues and trace every state and ledger event.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Complete Payment Lifecycle and Ledger History Returned):**
  - **Given** payment `PAY-20261123-0042` transitioned through `Initiated -> FundsHeld -> SentToBiller -> Unknown -> Completed`,
  - **When** an authenticated support employee searches by reference `PAY-20261123-0042`,
  - **Then** the system returns the payment details, biller reference, chronological state transitions with timestamps, associated ledger entries (hold and final charge), and notification delivery statuses.

- **Scenario 2 (Failure/Edge Case — Payment Reference Not Found):**
  - **Given** no payment record exists for reference `PAY-99999999-0000`,
  - **When** a support employee searches for `PAY-99999999-0000`,
  - **Then** the system returns a `404 Not Found` error with a clear message.

---

### FR-21: Block and Unblock Accounts (`Should`)

**User Story:** As a support employee, I want to block and unblock customer or agent accounts, so that suspicious accounts can be frozen immediately and restored once verified.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Block Account Prevents Payments and Logs Audit):**
  - **Given** an active customer account `USR-105`,
  - **When** an authenticated support employee blocks `USR-105` with a documented reason,
  - **Then** `USR-105` transitions to `blocked`,any subsequent payment attempt transitions Initiated -> Rejected during fraud checks, wallet top-ups and verification upgrades are rejected, and the block action is recorded in the audit log with the support employee's ID, timestamp, and reason.

- **Scenario 2 (Happy Path — Unblock Account Restores Payments and Logs Audit):**
  - **Given** a blocked customer account `USR-105`,
  - **When** an authenticated support employee unblocks `USR-105` with a documented reason,
  - **Then** `USR-105` transitions to `active`, subsequent valid payment attempts pass the blocked-account fraud check, and the unblock action is recorded in the audit log with the support employee's ID, timestamp, and reason.

- **Scenario 3 (Failure/Edge Case — Redundant Block Request Rejected):**
  - **Given** account `USR-105` is already in `blocked` status,
  - **When** a support employee submits another block request for `USR-105`,
  - **Then** the system rejects the request with a validation error and leaves the account state unchanged.

---

### FR-22: Payment Completion and Failure Notifications (`Must`)

**User Story:** As a customer, I want to receive both an SMS and a push notification when my payment completes or fails, so that I am promptly informed of the outcome and any failure reason.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — SMS and Push Sent on Completion):**
  - **Given** a customer's `352.50 EGP` ElectroCo payment (`PAY-020`) transitions to `Completed`,
  - **When** the payment reaches `Completed`,
  - **Then** PayLink dispatches both an SMS and a push notification containing `PAY-020`, `352.50 EGP`, and the `Completed` status.

- **Scenario 2 (Happy Path — SMS and Push Sent on Failure with Reason):**
  - **Given** a customer's `202.50 EGP` AquaCo payment (`PAY-021`) is refused by AquaCo with error code `ERR_AMOUNT_EXCEEDED` and transitions `Failed -> Refunded`,
  - **When** the payment fails and the hold is released,
  - **Then** PayLink dispatches both an SMS and a push notification containing `PAY-021`, `202.50 EGP`, the failed/refunded status, and a clear explanation of the refusal reason.

- **Scenario 3 (Failure/Edge Case — Notification Failure Never Blocks or Reverses Payment):**
  - **Given** ElectroCo confirms a `352.50 EGP` payment as `Completed` while the external SMS and push providers are unavailable,
  - **When** notification delivery fails,
  - **Then** the payment remains permanently `Completed` with the `352.50 EGP` charge finalized, and the failed notifications are queued for retry without rolling back the payment.

---

### FR-23: Notification Retry and Undelivered Marking (`Should`)

**User Story:** As a system operations engineer, I want failed notifications (SMS, push, or email) to be retried 3 times before being marked undelivered, so that transient messaging failures are recovered without impacting payment finality.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Transient SMS Failure Succeeded on Retry):**
  - **Given** an SMS notification for a completed payment fails on initial dispatch due to a transient gateway timeout,
  - **When** the retry worker re-attempts delivery and succeeds on retry attempt `2` of `3`,
  - **Then** the notification status transitions to `delivered` and records `2` retry attempts.

- **Scenario 2 (Failure/Edge Case — SMS Marked Undelivered After 3 Failed Retries):**
  - **Given** the SMS provider remains unreachable across the initial dispatch and all `3` retry attempts for a completed payment,
  - **When** the `3rd` retry fails,
  - **Then** PayLink marks the SMS notification as `undelivered`, stops further retries, logs the failure, and leaves the associated `Completed` payment unchanged.

- **Scenario 3 (Failure/Edge Case — Email Marked Undelivered After 3 Failed Retries):**
  - **Given** the email provider remains unreachable across the initial dispatch and all `3` retry attempts for a completed payment,
  - **When** the `3rd` retry fails,
  - **Then** PayLink marks the email notification as `undelivered`, stops further retries, logs the failure, and leaves the associated `Completed` payment unchanged.

---

### FR-24: Dynamic Configuration of Fees, Limits, and Promotions (`Should`)

**User Story:** As an admin, I want to edit fee values, transaction limits, and promotion schedules without redeploying the application, so that PayLink's business rules can be updated safely at runtime.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Updated Fee and Limit Applied to Next Initiated Payment Without Redeployment):**
  - **Given** an authenticated admin saves a configuration update changing the ElectroCo fee from `2.50 EGP` to `3.00 EGP` and the `basic` daily limit from `5,000.00 EGP` to `6,000.00 EGP`,
  - **When** the next ElectroCo bill inquiry or payment is initiated after the save (without restarting or redeploying the application),
  - **Then** the system applies the `3.00 EGP` fee and `6,000.00 EGP` daily limit to that new request, previously completed payments retain their original recorded fees, and the configuration change is written to the audit log.

- **Scenario 2 (Failure/Edge Case — Negative Fee Rejected):**
  - **Given** an authenticated admin submits an ElectroCo fee update of `-1.00 EGP`,
  - **When** the configuration update is validated,
  - **Then** the system rejects the request with a validation error and preserves the existing `2.50 EGP` fee.

- **Scenario 3 (Failure/Edge Case — Inverted Single-Payment Bounds Rejected):**
  - **Given** an authenticated admin submits single-payment bounds where minimum exceeds maximum (`min: 500.00 EGP`, `max: 100.00 EGP`),
  - **When** the configuration update is validated,
  - **Then** the system rejects the request with a validation error and preserves the existing bounds.

- **Scenario 4 (Failure/Edge Case — Inverted Promotion Window Rejected):**
  - **Given** an authenticated admin submits a promotion schedule with an end timestamp earlier than its start timestamp,
  - **When** the configuration update is validated,
  - **Then** the system rejects the request with a validation error and preserves the existing promotion configuration.

---

### FR-25: Comprehensive Audit Logging (`Must`)

**User Story:** As a compliance officer, I want every money movement and administrative or support action to be recorded in an immutable audit log, so that PayLink provides complete financial and operational auditability.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Money Movement Audited):**
  - **Given** an active customer initiates and completes a `350.00 EGP` ElectroCo payment (`2.50 EGP` fee, `352.50 EGP` total),
  - **When** the hold, final charge, and fee-income ledger entries commit,
  - **Then** an append-only audit log record is created for each money movement containing the actor (`who`), timestamp (`when`), action type, target entity ID, monetary amount, and reason (`why`).

- **Scenario 2 (Happy Path — Administrative Action Audited):**
  - **Given** an authenticated admin disables the AquaCo biller in the catalog with a maintenance reason,
  - **When** the biller status update commits,
  - **Then** an append-only audit log record is created containing the admin's ID (`who`), timestamp (`when`), action type, target biller ID, state change, and reason (`why`).

- **Scenario 3 (Failure/Edge Case — Audit Write Failure Rolls Back Money Movement):**
  - **Given** an active customer submits a `500.00 EGP` wallet top-up and the audit log write fails due to a database error,
  - **When** the system attempts to commit the top-up transaction,
  - **Then** the enclosing atomic database transaction rolls back completely, no ledger entry is committed, the wallet balance remains unchanged, and an error is returned so zero un-audited money movements occur.

---

### FR-26: Daily Settlement Reconciliation Report (`Should`)

**User Story:** As a finance operations specialist, I want PayLink to produce a daily reconciliation report from each biller's settlement file, so that `Unknown` payments are resolved and mismatches with final-state payments are flagged for support review.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Resolving `Unknown` Payments and Matching Settled Payments):**
  - **Given** ElectroCo's daily settlement file includes `PAY-001` (already `Completed` in PayLink) and `PAY-002` (stuck in `Unknown` after exhausted retries), while `PAY-003` (also in `Unknown`) is absent from the settlement file,
  - **When** the nightly reconciliation job processes the settlement file,
  - **Then** `PAY-001` is recorded as matched, `PAY-002` transitions `Unknown -> Completed` (converting the hold into a final charge), and `PAY-003` transitions `Unknown -> Failed -> Refunded` (releasing the held funds back to the customer's available balance).

- **Scenario 2 (Failure/Edge Case — Settlement File Lists a `Failed` Payment):**
  - **Given** ElectroCo's settlement file lists `PAY-004` (`350.00 EGP`) as received, whereas PayLink holds `PAY-004` in `Failed`,
  - **When** the nightly reconciliation job processes the settlement file,
  - **Then** PayLink flags `PAY-004` in the daily reconciliation discrepancy report for support review, does not re-transition `PAY-004` out of `Failed`, and allows support to log an auditable manual ledger adjustment if required.

- **Scenario 3 (Failure/Edge Case — Settlement File Lists a `Refunded` Payment):**
  - **Given** ElectroCo's settlement file lists `PAY-005` (`200.00 EGP`) as received, whereas PayLink already finalized `PAY-005` in `Refunded`,
  - **When** the nightly reconciliation job processes the settlement file,
  - **Then** PayLink flags `PAY-005` in the daily reconciliation discrepancy report for support review and leaves `PAY-005` permanently in `Refunded` without mutating the ledger automatically.

- **Scenario 4 (Failure/Edge Case — Amount Mismatch on a `Completed` Payment):**
  - **Given** ElectroCo's settlement file lists `PAY-006` as `360.00 EGP`, whereas PayLink holds `PAY-006` in `Completed` for `350.00 EGP`,
  - **When** the nightly reconciliation job processes the settlement file,
  - **Then** PayLink flags the `10.00 EGP` amount discrepancy in the daily reconciliation report for support review and leaves `PAY-006` in `Completed` without automatic ledger modification.

---

### FR-27: Post-Completion Refund Request and Support Approval (`Could`)

**User Story:** As a customer, I want to request a refund for a completed payment that a support employee can review and approve, so that disputed or erroneous completed payments can be reimbursed through an auditable workflow.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Customer Wallet Payment Refund Approved):**
  - **Given** a customer paid `PAY-010` (`350.00 EGP` bill + `2.50 EGP` fee = `352.50 EGP`) from their own wallet (`147.50 EGP` remaining), `PAY-010` is in `Completed`, and the customer submits `RefundRequest` (`REF-001`),
  - **When** an authenticated support employee approves `REF-001`,
  - **Then** `PAY-010` remains permanently in `Completed`, `REF-001` transitions to `Approved`, a compensating credit ledger entry of `352.50 EGP` is posted to the customer's wallet (restoring available balance to `500.00 EGP`), and an audit log entry is written.

- **Scenario 2 (Happy Path — Agent-Assisted Payment Refund Reverses Commission):**
  - **Given** an agent paid `PAY-011` (`350.00 EGP` bill + `2.50 EGP` fee = `352.50 EGP`, earning `1.00 EGP` commission; agent balance moved from `2,000.00 EGP` to `1,648.50 EGP`) from the agent's wallet, `PAY-011` is in `Completed`, and `RefundRequest` (`REF-002`) is submitted,
  - **When** an authenticated support employee approves `REF-002`,
  - **Then** `PAY-011` remains permanently in `Completed`, `REF-002` transitions to `Approved`, a compensating credit of `352.50 EGP` is posted to the agent's wallet, a `1.00 EGP` commission reversal debit is posted to the agent's wallet (restoring available balance to `2,000.00 EGP`), and an audit log entry is written.

- **Scenario 3 (Failure/Edge Case — Already-Refunded Payment Rejected to Prevent Double Refund):**
  - **Given** payment `PAY-012` is in final state `Refunded`,
  - **When** a customer submits a `RefundRequest` for `PAY-012`,
  - **Then** the system rejects the request with an invalid-state validation error and creates no ledger entry.

- **Scenario 4 (Failure/Edge Case — Non-Completed Lifecycle States Rejected):**
  - **Given** a payment is in a non-`Completed` state from Section 2.5 (tested individually across `Initiated`, `FundsHeld`, `SentToBiller`, `Unknown`, `Failed`, and `Rejected`),
  - **When** a customer submits a `RefundRequest` for that payment,
  - **Then** the system rejects the request with a validation error requiring state `Completed` and creates no ledger entry.

- **Scenario 5 (Failure/Edge Case — Duplicate Refund Request on Completed Payment Rejected):**
  - **Given** `Completed` payment `PAY-010` already has an `Approved` `RefundRequest` (`REF-001`),
  - **When** a second `RefundRequest` is submitted for `PAY-010`,
  - **Then** the system rejects the request with a `409 Conflict` error and creates no additional ledger entry.

---

### FR-28: Admin Daily Financial and Operational Totals (`Could`)

**User Story:** As an admin, I want to view daily totals for completed payments, service fees earned, commissions paid, and payment failures per biller, so that I can track daily financial performance and biller reliability.

**Acceptance Criteria:**

- **Scenario 1 (Happy Path — Daily Totals Aggregated by Cairo Calendar Day):**
  - **Given** on `2026-11-23` (`Africa/Cairo`) PayLink completed `120` payments totaling `42,000.00 EGP` in principal, earned `310.50 EGP` in service fees, paid `48.00 EGP` in agent commissions, and recorded `3` failures for ElectroCo and `7` failures for AquaCo,
  - **When** an authenticated admin requests the daily totals report for `2026-11-23`,
  - **Then** the system returns those exact decimal totals (`120` completed payments, `42,000.00 EGP` volume, `310.50 EGP` fees earned, `48.00 EGP` commissions paid) along with the per-biller failure breakdown (`ElectroCo: 3`, `AquaCo: 7`).

- **Scenario 2 (Failure/Edge Case — Date With Zero Transactions):**
  - **Given** no payments were processed on `2026-11-24` (`Africa/Cairo`),
  - **When** an authenticated admin requests the daily totals report for `2026-11-24`,
  - **Then** the system returns `0` completed payments, `0.00 EGP` volume, `0.00 EGP` fees earned, `0.00 EGP` commissions paid, and `0` failures per biller without error.

---

## 5. Non-Functional Requirements

| ID | Quality | Requirement Statement | Metric & Target | Measurement Method | Related Hard Cases (Sec. 2.7) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **NFR-1** | **Correctness** | Wallet and payment processing must prevent duplicate charges and negative balances under concurrent requests, network retries, and duplicate biller callbacks using ledger-derived balances and exact decimal math (`ROUND_HALF_UP` to 2 decimal places). | `0` double charges and `0` negative balances in concurrency and duplicate tests; *(Team-proposed)* `0` rounding discrepancies against exact decimal test vectors. | Automated Jest concurrency and idempotency integration tests firing parallel requests and duplicate callbacks against the NestJS app and relational DB in GitHub Actions CI. | `#1`, `#4`, `#6`|
| **NFR-2** | **Performance** | The payment backend must sustain high-volume bill payment throughput across the integrated mock billers during peak billing periods without dropping requests. | `>= 100` payments/second sustained for `1 minute` against the mock billers (ElectroCo, AquaCo, TopUpNet mix); *(Team-proposed)* `p95 <= 500 ms` on fast mock endpoints (ElectroCo sync payment and TopUpNet initial `"accepted"` response) and `< 1%` unexpected `5xx` error rate. | Load test script using `k6` (or `Artillery`) executed locally against the full `docker compose` stack (NestJS API, relational DB, and all three mock billers) | N/A *(stress-tests concurrent wallet handling alongside `#4`)* |
| **NFR-3** | **Resilience** | Failures, timeouts, or outages in one external biller or notification provider must remain isolated so payments to other active billers continue uninterrupted and completed payments are never rolled back. | `0%` of payments to healthy billers stopped or failed during another biller's outage; `100%` of live biller calls enforce the `10 s` timeout (`Unknown` state). | Jest fault-injection integration tests running in CI where one mock biller simulates a `12 s` timeout or error outage and the SMS mock returns errors while concurrent payments to healthy billers are asserted to reach `Completed`. | `#2`, `#3`, `#5` |
| **NFR-4** | **Modifiability** | Adding a new utility biller or updating an existing biller's protocol must be isolated to biller adapters and configuration without altering core payment, wallet, catalog, fee, or notification modules. | `0` files modified outside the `Biller Integration` module and configuration when adding a new biller or applying the ElectroCo API v2 change. | Git diff inspection (`git diff --stat`) on the PR introducing a new biller and on the Phase 2 ElectroCo v2 change-request PR, verified during PR review and reported in Phase 2. | `#8` |
| **NFR-5** | **Auditability** | Every money movement (top-up, hold, final charge, release/refund, commission, manual adjustment) and admin/support action must be recorded with actor, timestamp, and reason. | `100%` of money movements have an audit record capturing `who`, `when`, and `why`; *(Team-proposed)* `0` `UPDATE`/`DELETE` mutations permitted on the audit log table (strictly append-only). | Automated Jest database-integrity integration tests in CI verifying that every ledger entry has a matching, non-null audit record (`who`, `when`, `why`) and that every admin/support action creates a complete audit record without requiring a ledger row. | `#1`, `#3`, `#6`, `#7` |
| **NFR-6** | **Testability** | Domain logic across all modules must be decoupled from external I/O so business rules, state transitions, and failure scenarios are automatically verified in CI. | `>= 70%` line coverage on domain logic measured in CI; `8/8` (`100%`) of Section 2.7 hard cases covered by automated tests; *(Team-proposed)* CI pipeline fails automatically if domain coverage drops below `70%`. | Jest coverage report (`jest --coverage` with `coverageThreshold` scoped to domain services/entities) executed automatically in GitHub Actions on every push and PR. | `#1`–`#8` *(all hard cases)* |

### Measurement Notes

- **NFR-1 (Correctness):** Run a Jest integration suite inside GitHub Actions using `Promise.all` to fire concurrent wallet debits (`#4`), repeated identical idempotency keys (`#1`), payload-mismatch retries returning `409 Conflict` (D5), and duplicate TopUpNet callbacks (`#6`), then query the relational DB ledger sum to assert that no wallet balance is negative and each payment settled at most once.
- **NFR-2 (Performance):** Execute a 60-second constant-arrival-rate `k6` script (`100 payment requests/s`) locally against the `docker compose` stack using a realistic biller traffic mix across ElectroCo (`< 1 s`), AquaCo (`2–4 s`), and TopUpNet (immediate async `"accepted"`). Archive the summary output in the Phase 2 report to verify `>= 100 payments/s` across the combined mix, `< 1%` unexpected `5xx` errors, and the *(Team-proposed)* `p95 <= 500 ms` bound specifically on the fast ElectroCo and TopUpNet initial-accept endpoints.
- **NFR-3 (Resilience):** Configure mock biller fault modes via test control endpoints (ElectroCo `12 s` hang, AquaCo XML refusal, SMS `503` outage) in CI integration tests. Assert that payments routed to unaffected billers reach `Completed` with `0%` peer-induced failures, timed-out payments transition to `Unknown` with funds held, and failed SMS messages queue for `3` retries without rolling back `Completed` payments.
- **NFR-4 (Modifiability):** Enforce module isolation via PR review and inspect `git diff --name-only` to verify that adding a new biller adapter (`#8`) or implementing ElectroCo API v2 modifies only files inside the `Biller Integration` module (plus its adapter tests) and configuration, with zero diffs in any other module directory.
- **NFR-5 (Auditability):** Run an end-to-end Jest audit verification test in CI that executes top-ups, payments, refunds, commissions, reconciliation adjustments, and admin/support mutations. Query the database to assert that `100%` of ledger rows have a matching audit record (`actor_id`, `created_at`, `reason`) and that every admin/support action without a ledger row also writes a complete audit record.
- **NFR-6 (Testability):** Configure Jest in the NestJS repository to generate LCOV/text coverage summaries targeting domain modules and fail the GitHub Actions workflow if domain line coverage drops below `70%` or any of the `8` hard-case tests fail.