# Test Strategy & Quality Assurance Plan: Vighnesh AgroCare

## 1. Testing Philosophy & Governance

Agricultural retail involves real financial transactions, tight margins, and volatile inventory demands during peak sowing seasons. Faulty stock reservations or inaccurate credit ledgers directly impact farmer livelihoods and store profitability.

> **CRITICAL VERIFICATION RULE**:
> Tests are **only reported as passing after they have actually run and succeeded** in an active test execution environment. In this initial documentation and architecture phase (Phase 1), **no test suites exist or have been executed**.

---

## 2. Test Pyramid & Tooling

```
                     / \
                    /   \
                   / E2E \       Playwright
                  /-------\      (Browser flows, booking, POS checkout)
                 /  Integ  \     React Testing Library + Supabase SQL/pgTAP
                /-----------\    (Component integration & Database Stored RPCs)
               /  Unit Tests \   Vitest
              /---------------\  (WAC math, pricing, validation, i18n formatters)
```

### 2.1 Unit Tests (Vitest)
- **Scope**: Pure logic, mathematical calculations, string formatters, and utility libraries.
- **Coverage Targets**:
  - Weighted Average Cost (WAC) formula under varying purchase quantities and price swings.
  - Tax-inclusive price separation into base taxable value and GST breakdown.
  - Indian currency formatters (`₹1,25,000.00`) and Telugu/English date formatters.
  - Cart calculation logic (quantities, item subtotals, weight-based delivery estimates).

### 2.2 Integration Tests (React Testing Library & Supabase pgTAP)
- **Frontend Component Integration**:
  - Form validation states (preventing submission when stock is exceeded or required fields are missing).
  - Cart state persistence across mock page reloads.
  - Language toggle switching string dictionaries seamlessly.
- **Database Stored RPC & RLS Integration (pgTAP / SQL Test Runner)**:
  - Direct execution of `create_booking`, `receive_purchase`, `complete_sale`, `adjust_stock`.
  - Verification of constraint triggers (`selling_price <= mrp`, `available_stock >= 0`).
  - Row Level Security policy testing: confirming that users with `employee` role receive permission errors when querying purchase costs or supplier records.

### 2.3 End-to-End Tests (Playwright)
- **Customer Journey**:
  - Browse bilingual catalog in Telugu -> add products to cart -> proceed to checkout -> submit booking -> verify 24h timer and WhatsApp redirect link.
- **Store Staff Counter Journey**:
  - Login as employee -> open POS screen -> lookup booking -> convert to counter sale -> record Cash payment -> print receipt.
- **Admin Workflow**:
  - Login as admin -> ingest goods receipt -> verify inventory count and WAC update -> check profit reports.

---

## 3. Critical Edge Cases & Failure Modes

The test suite must specifically target and validate the following 9 mission-critical edge cases:

### EC-01: Insufficient Stock
- **Scenario**: A customer attempts to add or book 5 units of an item when `available_stock` is only 2.
- **Expected Behavior**: The UI blocks incrementing beyond available stock. If bypassed, the `create_booking` stored function raises an exception: `"Insufficient available stock"` and rolls back the transaction entirely.

### EC-02: Simultaneous Bookings (Race Condition)
- **Scenario**: Two farmers attempt to book the final remaining bottle of a high-demand pesticide at the exact same millisecond.
- **Expected Behavior**: PostgreSQL `SELECT ... FOR UPDATE` row locks force serial execution. The first transaction succeeds and reserves the bottle. The second transaction immediately encounters `available_stock = 0`, is rejected cleanly with an out-of-stock message, and no negative stock is created.

### EC-03: Duplicate Submission (Double Clicks / Retries)
- **Scenario**: A user clicks "Confirm Booking" multiple times or the client automatically retries due to a momentary packet drop.
- **Expected Behavior**: The request carries a client-generated UUID `idempotency_key`. The backend detects the existing key, returns the original booking confirmation without reserving duplicate stock or inserting extra rows.

### EC-04: Cancelled & Expired Booking Lifecycle
- **Scenario**: A booking is cancelled by the customer, or 24 hours elapse without store collection.
- **Expected Behavior**: The `cancel_booking` function or scheduled `expire_stale_bookings` cron job decrements `reserved_stock`, restores `available_stock`, sets status to `cancelled`, and records an immutable `reservation_release` ledger row.

### EC-05: Duplicate Purchase Goods Receipt
- **Scenario**: Staff double-submits a supplier goods receipt or browser reloads during receipt processing.
- **Expected Behavior**: The purchase receipt requires a unique supplier invoice number and idempotency token. Re-submission is rejected, preventing accidental double-counting of physical inventory or inaccurate WAC inflation.

### EC-06: Duplicate Counter Sale Submission
- **Scenario**: A POS counter sale is triggered twice during rapid checkout or payment barcode scanning.
- **Expected Behavior**: Transaction idempotency validates payment and sale creation; duplicate transaction requests fail or return the existing invoice record without double-deducting physical stock.

### EC-07: Unauthorized Database Access Attempt
- **Scenario**: A malicious user or an employee attempts a direct Supabase PostgREST query to view supplier purchase prices or owner profit reports.
- **Expected Behavior**: PostgreSQL Row Level Security (RLS) evaluates the request context (`auth.uid()` and role). The query returns an empty result set or raises an explicit 403 Forbidden error.

### EC-08: Invalid Payment Transitions
- **Scenario**: A sale is recorded with an invalid payment amount (e.g. negative value, total payments less than invoice total without credit approval, or credit granted exceeding the customer's credit limit).
- **Expected Behavior**: The `complete_sale` transaction validates payment sums and credit limits inside the database boundary and rolls back if criteria are not strictly met.

### EC-09: Intermittent Network Drops (Rural Connectivity)
- **Scenario**: The farmer's cellular signal drops immediately after pressing "Book Now".
- **Expected Behavior**: The PWA gracefully alerts the user of the network disconnection without discarding form data. When connectivity resumes, the idempotent request can be safely retried without risk of double reservation.

---

## 4. Test Reporting Standards

- No test execution claims are made during project documentation phases.
- CI/CD pipelines (GitHub Actions) will enforce that all Vitest unit suites, Supabase SQL tests, and Playwright end-to-end tests run on pull requests to `main`.
- Pull requests cannot be merged if any test fails or if coverage drops below agreed project baselines.
