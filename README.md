# The Mathematics of Financial Ledger Integrity

### Designing accounting systems that prevent double-spending within a controlled ledger

**Reference architecture | Financial engineering · Atomic transactions · Concurrency control · Double-entry accounting**

**Author:** Shavez Ahmed Siddiqui  
**Technical research:** https://shavezahmedsiddiqui.com/  
**Status:** Educational specification and executable reference code — **not a deployed, audited, or production-ready banking system**.

---

## Overview

A financial system is correct only when **every committed transaction preserves its accounting and authorization invariants**. Fast APIs, message delivery, and double-entry postings are useful but insufficient individually: two balanced transfers may still overspend an account unless available-funds validation occurs under an appropriate concurrency-control boundary.

This repository translates the Substack article *The Mathematics of Financial Ledger Integrity: Designing Accounting Systems That Cannot Double-Spend* into a developer-oriented reference specification. The implementation demonstrates **single-database, atomic internal transfers** with durable reservations, journal balancing, idempotency, and concurrent-spend rejection.

> **Scope:** The example establishes properties only for financial state controlled by one transactional database. It does **not** claim universal double-spend prevention across externally controlled wallets, independent payment networks, cross-currency settlement systems, or compromised infrastructure.

## Repository map

```text
financial-ledger-integrity-spec/
├── README.md
├── docs/
│   ├── ACCOUNTING_MODEL.md
│   ├── CONCURRENCY_AND_IDEMPOTENCY.md
│   ├── RESERVATIONS_AND_STATE.md
│   └── TESTING_AND_LIMITS.md
├── examples/
│   ├── journal.json
│   ├── transfer.json
│   └── reservation.json
├── src/
│   ├── __init__.py
│   └── ledger.py
├── tests/
│   └── test_ledger.py
└── .gitignore
```

## 1. Architecture: control before financial effect

<img width="2050" height="1954" alt="mermaid-diagram (19)" src="https://github.com/user-attachments/assets/e94d5c64-00d7-40cd-b97f-aa84f981a60b" />

**Transaction boundary:** The posting set, account balance projections, and journal identity must commit or roll back together. The example uses a single database write transaction for these changes. Any external settlement operation must be managed through an independent state machine and reconciliation process.

## 2. Financial invariants

Let \(S_t\) be the financial state and \(T\) a proposed state transition. A transaction is admissible only if its committed state satisfies the constraints below.

### 2.1 Balanced journal, per accounting unit

For each journal \(j\) and currency \(c\):

<img width="845" height="95" alt="Screenshot 2026-11-13 170905" src="https://github.com/user-attachments/assets/a34b0076-590a-415f-a479-2463e82c4734" />

A currency conversion is **not** expressed by offsetting 20 units of one asset against 100 units of another as though their numerical values were equivalent. Currency-specific legs need the relevant clearing, inventory, valuation, or obligation accounts.

### 2.2 Nonnegative spendable balances

For an account not permitted to overdraw:

<img width="855" height="70" alt="Screenshot 2026-11-13 171010" src="https://github.com/user-attachments/assets/98ac07b5-5a79-4f26-aacb-2478434de7f5" />

- \(P_a\): eligible posted balance, represented in integer minor units.
- \(H_a\): active reserved amount.
- \(A_a\): available amount.

An application may expose different balance classes and business-specific credit limits; those rules must be included in the actual invariant being enforced.

### 2.3 Exactly one financial effect per instruction

For instruction identity \(k\), repeating a *matching* committed request must not create new postings:

<img width="797" height="75" alt="Screenshot 2026-11-13 170839" src="https://github.com/user-attachments/assets/c96d3449-6c71-46d7-99b0-5185ffe3231d" />

A reused identity with different parameters is a **conflict**, not a valid retry. This is a guarantee of the internal ledger logic, not a claim of exactly-once delivery across networks.

### 2.4 Reservation and journal consistency

For each controlled account:

<img width="861" height="76" alt="Screenshot 2026-11-13 171147" src="https://github.com/user-attachments/assets/348c3101-e410-4114-917d-c816bc7b267a" />

For each account \(a\), the materialized balance must equal the signed projection of its committed journal postings. A hold is an availability control, **not** a completed financial posting.

Further detail: [Accounting model](docs/ACCOUNTING_MODEL.md).

## 3. Why concurrency matters

Assume the available balance is 100 units. Two requests attempt to transfer 80 and 70 units simultaneously. Both are affordable independently; they are not jointly affordable.

A naive check-then-write pattern is unsafe:

```python
balance = read_balance(account)
if balance >= amount:
    write_balance(account, balance - amount)  # unsafe under concurrent requests
```

The authoritative ledger must perform the funds check and posting under a control that serializes or detects conflicting decisions.

The reference implementation begins a database write transaction *before* checking funds and commits the complete internal journal afterward. Competing writers cannot independently approve incompatible debits against the same starting balance.

See [Concurrency and idempotency](docs/CONCURRENCY_AND_IDEMPOTENCY.md).

## 4. Double-entry posting example

A transfer of 75.00 currency units between two **customer liability** accounts, represented as 7,500 minor units:

| Account | Debit | Credit | Normal balance effect |
|---|---:|---:|---|
| Sender liability | 7,500 | — | Decreases |
| Recipient liability | — | 7,500 | Increases |
| **Journal total** | **7,500** | **7,500** | **Balanced** |

A valid opening liability has a corresponding asset or other economic counterpart. In the tests, opening funds are represented by a balanced journal debiting a settlement asset account and crediting a customer liability account. No balances are silently inserted.

## 5. Reservations are not settlement

<img width="638" height="740" alt="mermaid-diagram (20)" src="https://github.com/user-attachments/assets/5c4dfd58-75f9-41e8-9f08-99d0c2565d8f" />

A reservation reduces **available** funds, not the posted accounting balance. Capturing a hold performs the associated journal transfer and consumes the hold in the **same local transaction**, preventing a double reduction of availability.

For assets outside this controlled ledger, a database reservation is only an application-side commitment unless an enforceable asset-control mechanism exists.

See [Reservations and state](docs/RESERVATIONS_AND_STATE.md).

## 6. Run the reference implementation

Requires **Python 3.10+**, with no third-party Python packages.

```bash
python -m unittest discover -s tests -v
```

Create a local balanced funding entry, reserve funds, and capture the reservation:

```python
from src.ledger import Ledger

ledger = Ledger("ledger_demo.db")
ledger.create_account("cash", "USD", "DEBIT")
ledger.create_account("payer", "USD", "CREDIT")
ledger.create_account("recipient", "USD", "CREDIT")

ledger.post_journal("opening-001", [
    {"account_id": "cash", "currency": "USD", "side": "DEBIT", "amount_minor": 10000},
    {"account_id": "payer", "currency": "USD", "side": "CREDIT", "amount_minor": 10000},
])

ledger.reserve("hold-001", "payer", 7500)
ledger.capture("hold-001", "recipient", "journal-001")
ledger.assert_invariants()

print(ledger.balance("payer"))
# posted_minor: 2500, reserved_minor: 0, available_minor: 2500
```

Use a new database file for each clean run, or handle preexisting accounts and journals. Amounts are represented as integer minor units, never floating-point currency values.

### Code contracts

| Operation | Purpose | Idempotency behavior |
|---|---|---|
| `post_journal(id, lines)` | Apply a balanced journal | Exact replay returns `False`; changed payload raises `Conflict` |
| `transfer(id, source, dest, minor)` | Internal transfer between same-currency customer liabilities | Exact replay returns `False` |
| `reserve(hold_id, account, minor)` | Protect an internal available balance | Repeated active reservation returns `False` |
| `release(hold_id)` | Release an active hold | Repeated release returns `False` |
| `capture(hold_id, dest, journal_id)` | Atomically consume hold and post transfer | Exact repeated capture returns `False` |
| `assert_invariants()` | Check journal, balance-projection, and hold consistency | Raises on mismatch |

These APIs are deliberately narrow: authorization, network permissions, cross-currency FX, and external settlement are **not** implemented.

## 7. Testing and expected properties

The test suite exercises:

- Journal debit-credit balancing and projection correctness
- Atomic internal account transfers
- Insufficient-funds rejection
- Idempotent retries and conflicting identifiers
- Reservation enforcement, capture, and release
- Rollback after a failed capture
- Cross-currency transfer rejection
- Integer-only amount validation
- Concurrent transfers competing for the same balance
- Concurrent holds competing for the same available funds

A concurrency fixture funds a source account with 10,000 minor units. Twenty-four concurrent requests attempt to transfer 1,000 units each. **Exactly ten** must commit; the remainder are rejected because funds are exhausted. The test checks the closing balances and ledger invariants.

Passing tests demonstrate these particular scenarios only. They do not prove correctness under every crash, network, database configuration, or adversarial threat. See [Testing and limitations](docs/TESTING_AND_LIMITS.md).

## 8. External settlement boundary

<img width="2224" height="1518" alt="mermaid-diagram (21)" src="https://github.com/user-attachments/assets/f2949713-3bed-49c5-afd9-a2dea5560b3a" />

This diagram describes a **design boundary**, not implemented external-network logic. The illustrative `src/ledger.py` only handles balances and journals inside one local database.

## 9. Non-goals and security posture

This reference does not implement customer identity verification, cryptographic authorization, fraud screening, credit limits, multi-entity accounting, FX inventory, asynchronous payment orchestration, high availability, secure secrets handling, or regulatory compliance controls. It is not a bank ledger, payment processor, wallet, or smart contract.

For actual financial workloads, the design needs independent security review, formal accounting specifications, role separation, operational controls, database recovery testing, and an approved settlement/reconciliation architecture.

## Author

**Shavez Ahmed Siddiqui** writes about financial infrastructure, ledger architecture, payment engineering, and distributed systems. Additional technical research: [shavezahmedsiddiqui.com](https://shavezahmedsiddiqui.com/).

*Technical/educational material only. This repository is a conceptual reference implementation and provides no claim of independent audit or production deployment.*
