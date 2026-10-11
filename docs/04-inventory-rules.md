# Inventory Accounting & Concurrency Rules: Vighnesh AgroCare

## 1. Stock Definitions & Accounting Model

Inventory balances in **Vighnesh AgroCare** are structured around three strictly defined values maintained per sellable SKU:

1. **`physical_stock`**:
   The actual, verified count of product units physically resting on the shelves, racks, or godown of the shop.
2. **`reserved_stock`**:
   The quantity of units committed to active, unexpired customer bookings (held for up to 24 hours). These units remain physically inside the store but are legally and contractually promised to a customer.
3. **`available_stock`**:
   The uncommitted quantity immediately available for counter sale or new customer bookings:
   $$\text{available\_stock} = \text{physical\_stock} - \text{reserved\_stock}$$

### Core Invariants
- $\text{physical\_stock} \ge 0$ (enforced by DB check constraint).
- $\text{reserved\_stock} \ge 0$ (enforced by DB check constraint).
- $\text{physical\_stock} \ge \text{reserved\_stock}$, therefore $\text{available\_stock} \ge 0$ at all times.
- No sale or reservation can be initiated if requested quantity exceeds $\text{available\_stock}$.

---

## 2. Append-Only Inventory Movements Ledger

All inventory alterations are governed by an immutable, append-only transaction ledger: `inventory_movements`.

### Ledger Schema Rules
- **No `UPDATE` or `DELETE`**: Database privileges on `inventory_movements` deny update and delete operations completely, even to administrators.
- **Auditing**: Every record stores `movement_id`, `product_id`, `movement_type`, `quantity`, `reference_id` (e.g. sale ID, booking ID, purchase ID), `reference_type`, `unit_cost_at_movement`, `created_at`, and `created_by`.

### Movement Types Specification

| Movement Type | Physical Stock Impact | Reserved Stock Impact | Description / Trigger |
| :--- | :---: | :---: | :--- |
| `purchase_receipt` | **+ (Increase)** | None | Ingestion of received stock from supplier invoice. |
| `sale` | **- (Decrease)** | **- (Decrease if fulfilling booking)** | Counter billing sale or fulfilled customer booking. |
| `reservation` | None | **+ (Increase)** | Creation of an online customer booking. |
| `reservation_release` | None | **- (Decrease)** | Cancellation or 24-hour timeout of a booking. |
| `adjustment_in` | **+ (Increase)** | None | Physical stock count audit finding excess stock. |
| `adjustment_out` | **- (Decrease)** | None | Physical stock count audit finding missing stock. |
| `damage` | **- (Decrease)** | None | Broken bottle, punctured packet, or chemical leakage. |
| `expiry` | **- (Decrease)** | None | Product surpassed manufacturer expiry date. |
| `customer_return` | **+ (Increase)** | None | Customer returns undamaged item with valid receipt. |
| `supplier_return` | **- (Decrease)** | None | Return of damaged/recalled goods to distributor. |

---

## 3. Booking Reservation Lifecycle

Online bookings lock stock for up to **24 hours** to prevent farmers from traveling to the shop only to find seasonal crop protection products sold out, while ensuring inventory does not remain blocked indefinitely.

```
       +-----------------------+
       |   Customer Initiates  |
       |      New Booking      |
       +-----------------------+
                   |
                   v
       [create_booking RPC]
       - Locks product_stock (FOR UPDATE)
       - Validates available_stock >= qty
       - reserved_stock += qty
       - Movement: reservation
                   |
                   v
       +-----------------------+
       |    Status: PENDING    |
       | (24-Hour Timer Active)|
       +-----------------------+
         /         |         \
        /          |          \
(Customer/Admin (24h Passes    (Customer Arrives at Counter
   Cancels)     Uncollected)     or Home Delivery Completed)
      /            |            \
     v             v             v
[cancel_booking] [Scheduled Cron] [complete_sale]
- reserved -= qty - reserved -= qty - physical -= qty
- Movement:       - Movement:       - reserved -= qty
  reservation_      reservation_    - Movement: sale
  release           release         - Invoice created
     |             |             |
     v             v             v
+-----------+ +-----------+ +-------------+
| Status:   | | Status:   | | Status:     |
| CANCELLED | | CANCELLED | | COMPLETED   |
+-----------+ +-----------+ +-------------+
```

### Lifecycle Stages
1. **Creation (`create_booking`)**:
   - Client sends product list with idempotency key.
   - Transaction executes `SELECT ... FOR UPDATE` on each product.
   - Verifies `available_stock >= requested_qty`.
   - Increments `reserved_stock`, creates booking with `status = 'pending'`, sets `expires_at = NOW() + INTERVAL '24 hours'`, and records `reservation` movement.
2. **Cancellation (`cancel_booking`)**:
   - If cancelled manually by customer or shop staff: decrements `reserved_stock`, marks booking as `cancelled`, and logs `reservation_release` movement.
3. **Automatic Expiry (`expire_stale_bookings`)**:
   - A scheduled cron job queries all bookings where `status = 'pending'` AND `expires_at < NOW()`.
   - Iterates through items, releases `reserved_stock`, sets status to `cancelled`, and logs `reservation_release`.
4. **Fulfillment & Sale Completion (`complete_sale`)**:
   - Customer collects goods and pays.
   - Decrements `physical_stock` by the purchased quantity.
   - Decrements `reserved_stock` by the purchased quantity.
   - Marks booking as `completed` and logs `sale` movement.

---

## 4. Concurrency Safety & Race Condition Prevention

High demand for seasonal pesticides during sudden pest attacks (e.g. pink bollworm outbreaks in cotton or brown planthopper attacks in paddy) can cause hundreds of simultaneous booking attempts when fresh consignments arrive.

### Concurrency Rules
1. **Row-Level Locking**: Every inventory mutation MUST obtain an exclusive row lock on `product_stock` using `SELECT ... FOR UPDATE` before reading or updating stock balances. This forces concurrent transactions into an orderly, serial queue.
2. **Transaction Isolation**: All read-check-write sequences execute inside atomic PostgreSQL transactions (`BEGIN ... COMMIT`).
3. **Idempotency Keys**:
   - Every state-altering call (`create_booking`, `receive_purchase`, `complete_sale`) takes a UUID `p_idempotency_key`.
   - Stored functions check an `idempotency_records` table. If the key exists, the original response is returned without re-executing stock movements.

---

## 5. Weighted Average Cost (WAC) Method

AgroCare tracks inventory valuation using the **Weighted Average Cost (WAC)** method. Because retail sales of agrochemicals occur rapidly over the counter, scanning specific batch purchase costs during rapid counter billing is operationally unfeasible.

### Definition & Formula
When new inventory arrives via a purchase receipt:
$$\text{New WAC} = \frac{(\text{Current Physical Stock} \times \text{Current WAC}) + (\text{New Purchase Quantity} \times \text{New Unit Cost})}{\text{Current Physical Stock} + \text{New Purchase Quantity}}$$

### Worked Step-by-Step Example

#### Scenario: Chlorpyrifos 20% EC (1 Litre Bottle)

#### Step 1: Initial State
- The shop currently holds **10 bottles** in stock.
- The recorded Current WAC is **₹400.00 per bottle**.
- Total inventory value = $10 \times 400 = \text{₹4,000.00}$.

#### Step 2: New Consignment Received
- The distributor delivers **20 bottles** on a fresh invoice.
- Due to market price changes, the new purchase price is **₹430.00 per bottle**.
- Additional inventory value = $20 \times 430 = \text{₹8,600.00}$.

#### Step 3: Recomputing Weighted Average Cost
- $\text{Total New Physical Stock} = 10 + 20 = 30\text{ bottles}$.
- $\text{Total Inventory Value} = ₹4,000.00 + ₹8,600.00 = ₹12,600.00$.
- $\text{New WAC} = \frac{₹12,600.00}{30} = \mathbf{₹420.00\text{ per bottle}}$.
- The product's recorded WAC in the database is automatically updated to **₹420.00**.

#### Step 4: Subsequent Retail Sale
- A local farmer purchases **6 bottles** at the counter at an MRP/selling price of **₹520.00**.
- Physical stock decreases from **30 to 24 bottles**.
- **WAC remains unchanged at ₹420.00 per bottle**.
- Remaining inventory valuation = $24 \times 420.00 = \text{₹10,080.00}$.
- Realized Gross Profit on sale = $6 \times (520.00 - 420.00) = \mathbf{₹600.00}$.
