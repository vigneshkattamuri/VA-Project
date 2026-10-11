# System Architecture: Vighnesh AgroCare

## 1. Architectural Strategy

**Vighnesh AgroCare** is structured as a high-performance single-page Progressive Web Application (PWA) with a headless, database-centric backend powered by Supabase and PostgreSQL, tailored specifically for a retail pesticide shop in Andhra Pradesh, India.

### Core Architectural Principles
1. **Single Frontend Application with Segmented Layouts**: A unified React + TypeScript codebase serves both public customers (farmers) and internal store operators via distinct route hierarchies and layouts:
   - Customer Portal (`/`): Lightweight, mobile-first, bilingual (English/Telugu), optimized for fast browsing of crop protection products, cart management, and booking.
   - Admin & Staff Portal (`/admin`): Desktop/tablet-optimized, counter sales and billing, inventory audit, goods receipt, delivery management, and reports.
2. **Database-Centric Business Logic (PL/pgSQL Functions)**: All critical business transactions—such as stock reservations, booking transitions, purchase receipts, inventory adjustments, counter sales billing, and payment processing—execute exclusively inside PostgreSQL stored functions. The frontend browser **never** writes directly to inventory or financial tables.
3. **Concurrency Safety & ACID Guarantees**: Stock deduction and booking reservation utilize explicit row locking (`SELECT ... FOR UPDATE`), transaction isolation, and database-level idempotency checks.
4. **Minimal Edge Functions**: Supabase Edge Functions (TypeScript/Deno) are utilized solely when server-side secrets or third-party webhooks are involved (e.g. CAPTCHA verification, future SMS gateway integration). Public and authenticated database interactions occur via PostgREST and RPC calls governed by strict Row Level Security (RLS).
5. **Strict Environment Isolation**: Independent development and production Supabase projects to safeguard financial and inventory integrity.

---

## 2. Text Architecture Diagram

```
+-----------------------------------------------------------------------------------------+
|                                    CLIENT TIER (PWA)                                    |
|                                                                                         |
|   +---------------------------------------+   +---------------------------------------+ |
|   |         Customer Interface            |   |           Admin & Staff Portal        | |
|   |      (Mobile PWA, English/Telugu)     |   |         (/admin Layout & Views)       | |
|   |                                       |   |                                       | |
|   |   - Pesticide Catalog & Search        |   |   - Counter Sales & Billing           | |
|   |   - Cart & Checkout                   |   |   - Goods Receipt & Purchases         | |
|   |   - 24h Booking Creation              |   |   - Inventory Adjustments             | |
|   |   - WhatsApp Integration (wa.me)      |   |   - Udhaar Credit Ledger              | |
|   |   - Customer Profile                  |   |   - Delivery & Reports                | |
|   +---------------------------------------+   +---------------------------------------+ |
|                       |                                           |                     |
|                       +---------------------+---------------------+                     |
|                                             |                                           |
|                                [HTTPS / WSS (Realtime)]                                 |
+---------------------------------------------|-------------------------------------------+
                                              |
+---------------------------------------------v-------------------------------------------+
|                                   INFRASTRUCTURE TIER                                   |
|                                                                                         |
|   +-------------------------+                           +---------------------------+   |
|   |   Vercel Edge Network   |                           |    Cloudflare Turnstile   |   |
|   | (Static Host & CDN)     |                           |    (Bot / CAPTCHA Guard)  |   |
|   +-------------------------+                           +---------------------------+   |
|                                                                       |                 |
|   +-------------------------------------------------------------------v-------------+   |
|   |                         SUPABASE BACKEND CLOUD PLATFORM                         |   |
|   |                                                                                 |   |
|   |   +--------------------+  +--------------------+  +-------------------------+   |   |
|   |   |   Supabase Auth    |  |  Supabase Storage  |  | Supabase Edge Functions |   |   |
|   |   | (JWT, RBAC claims) |  | (Product Images)   |  | (Secrets, Bot Check)     |   |   |
|   |   +--------------------+  +--------------------+  +-------------------------+   |   |
|   |             |                       |                          |                |   |
|   |             +-----------------------+--------------------------+                |   |
|   |                                     |                                           |   |
|   |   +---------------------------------v---------------------------------------+   |   |
|   |   |                   PostgreSQL 15+ Database Engine                        |   |   |
|   |   |                                                                         |   |   |
|   |   |   +-----------------------------------------------------------------+   |   |   |
|   |   |   |                Row Level Security (RLS) Layer                   |   |   |   |
|   |   |   |  - Customer / Admin / Employee / Delivery Policies              |   |   |   |
|   |   |   |  - Read-Only access to stock levels; Direct writes FORBIDDEN    |   |   |   |
|   |   |   +-----------------------------------------------------------------+   |   |   |
|   |   |                                 |                                       |   |   |
|   |   |   +-----------------------------v-----------------------------------+   |   |   |
|   |   |   |       Core Business Logic Engine (PL/pgSQL Stored RPCs)         |   |   |   |
|   |   |   |                                                                 |   |   |   |
|   |   |   |   * create_booking(items, customer_info, idempotency_key)       |   |   |   |
|   |   |   |   * transition_booking(booking_id, new_status)                  |   |   |   |
|   |   |   |   * cancel_booking(booking_id, reason)                          |   |   |   |
|   |   |   |   * receive_purchase(supplier_id, invoice_meta, items)          |   |   |   |
|   |   |   |   * adjust_stock(product_id, qty, movement_type, notes)         |   |   |   |
|   |   |   |   * complete_sale(items, payments, customer_id, booking_id)     |   |   |   |
|   |   |   |   * record_payment(sale_id, customer_id, amount, mode)          |   |   |   |
|   |   |   |                                                                 |   |   |   |
|   |   |   |   [ACID Transactions | SELECT ... FOR UPDATE | Idempotency Keys]    |   |   |   |
|   |   |   +-----------------------------------------------------------------+   |   |   |
|   |   |                                 |                                       |   |   |
|   |   |   +-----------------------------v-----------------------------------+   |   |   |
|   |   |   |                     Data Persistence Layer                      |   |   |   |
|   |   |   |                                                                 |   |   |   |
|   |   |   |   - products (CIB&RC, active ingredient, formulation, toxicity) |   |   |   |
|   |   |   |   - product_stock (physical, reserved, WAC cost)                |   |   |   |
|   |   |   |   - inventory_movements (Immutable append-only ledger)          |   |   |   |
|   |   |   |   - bookings & booking_items (24h reservations)                 |   |   |   |
|   |   |   |   - sales & sale_items & sale_payments                          |   |   |   |
|   |   |   |   - customer_credit_ledger (Udhaar balance & limits)            |   |   |   |
|   |   |   |   - suppliers & purchase_orders & purchase_items                |   |   |   |
|   |   |   |   - shop_settings                                               |   |   |   |
|   |   |   +-----------------------------------------------------------------+   |   |   |
|   |   |                                                                         |   |   |
|   |   |   +-----------------------------------------------------------------+   |   |   |
|   |   |   |         Scheduled Jobs (pg_cron / Supabase Cron)                |   |   |   |
|   |   |   |   * expire_stale_bookings() (Releases 24h expired reservations) |   |   |   |
|   |   |   +-----------------------------------------------------------------+   |   |   |
|   |   +-------------------------------------------------------------------------+   |   |
|   +---------------------------------------------------------------------------------+   |
+-----------------------------------------------------------------------------------------+
```

---

## 3. Database-Centric Business Functions

The browser client is strictly restricted from performing direct SQL `INSERT`, `UPDATE`, or `DELETE` operations on core inventory, sales, and accounting tables. All state mutations occur via PostgreSQL functions executed with `SECURITY DEFINER` privileges, guaranteeing transaction boundaries and audit enforcement.

### 3.1 `create_booking`
- **Parameters**: `p_customer_id`, `p_customer_name`, `p_phone`, `p_delivery_type`, `p_delivery_address`, `p_items` (JSON array of `product_id`, `quantity`), `p_idempotency_key`.
- **Concurrency & Locking**:
  - Checks if `p_idempotency_key` already exists; returns existing booking if duplicate.
  - Loops through requested items and executes `SELECT physical_stock, reserved_stock FROM product_stock WHERE product_id = item.product_id FOR UPDATE`.
  - Verifies `(physical_stock - reserved_stock) >= item.quantity`. If insufficient, aborts with an explicit exception.
  - Updates `reserved_stock = reserved_stock + item.quantity`.
  - Inserts into `inventory_movements` with movement type `reservation`.
  - Inserts booking record with `status = 'pending'` and sets `expires_at = NOW() + INTERVAL '24 hours'`.

### 3.2 `transition_booking`
- **Parameters**: `p_booking_id`, `p_new_status`, `p_notes`.
- **Execution**: Validates allowed status transitions according to the state machine (e.g. `pending` -> `confirmed` -> `ready_for_pickup` -> `out_for_delivery` -> `completed`).

### 3.3 `cancel_booking`
- **Parameters**: `p_booking_id`, `p_cancellation_reason`.
- **Execution**:
  - Locks booking row `FOR UPDATE`.
  - Verifies booking is in a cancellable state (`pending`, `confirmed`, `ready_for_pickup`).
  - Iterates over booking items, locks `product_stock FOR UPDATE`, and decrements `reserved_stock = reserved_stock - item.quantity`.
  - Inserts `inventory_movements` record with movement type `reservation_release`.
  - Updates booking status to `cancelled`.

### 3.4 `receive_purchase`
- **Parameters**: `p_supplier_id`, `p_invoice_number`, `p_invoice_date`, `p_items` (JSON array of `product_id`, `quantity`, `cost_price_per_unit`, `batch_number`, `expiry_date`), `p_idempotency_key`.
- **Execution**:
  - Locks product stock rows `FOR UPDATE`.
  - For each product, computes New Weighted Average Cost:
    $$\text{New WAC} = \frac{(\text{Current Physical Stock} \times \text{Current WAC}) + (\text{Received Qty} \times \text{Purchase Cost})}{\text{Current Physical Stock} + \text{Received Qty}}$$
  - Increments `physical_stock = physical_stock + item.quantity`.
  - Updates product's current WAC.
  - Inserts movement into `inventory_movements` (`purchase_receipt`).
  - Records batch and expiry metadata in `purchase_items`.

### 3.5 `adjust_stock`
- **Parameters**: `p_product_id`, `p_quantity`, `p_movement_type` (`adjustment_in`, `adjustment_out`, `damage`, `expiry`), `p_reason_notes`.
- **Execution**:
  - Restricted to role `admin`.
  - Locks product stock `FOR UPDATE`.
  - For reductions (`adjustment_out`, `damage`, `expiry`), validates `physical_stock - p_quantity >= reserved_stock` to prevent stranding active reservations.
  - Adjusts `physical_stock` accordingly and logs `inventory_movements`.

### 3.6 `complete_sale`
- **Parameters**: `p_customer_id`, `p_booking_id` (optional), `p_items` (JSON array of `product_id`, `quantity`, `unit_price`), `p_payments` (JSON array of `mode`, `amount`), `p_idempotency_key`.
- **Execution**:
  - Checks idempotency key to prevent double billing.
  - Enforces `unit_price <= mrp` for all items.
  - Locks product stock `FOR UPDATE`.
  - If linked to a booking: decrements `physical_stock` and decrements `reserved_stock`.
  - If counter walk-in sale: validates `available_stock >= quantity`, then decrements `physical_stock`.
  - Inserts `sale` and line items.
  - Inserts `inventory_movements` (`sale`).
  - Records payments. If payment mode is `credit` (udhaar): validates customer outstanding balance + new credit <= `credit_limit`, then records entry in `customer_credit_ledger`.

### 3.7 `record_payment`
- **Parameters**: `p_customer_id`, `p_sale_id` (optional), `p_amount`, `p_payment_mode` (`cash`, `upi`), `p_reference_number`.
- **Execution**:
  - Records repayment towards outstanding credit balance in `customer_credit_ledger`.
  - Updates payment allocation and receipt details.

---

## 4. Planned Source Code Layout (Phase 2 Initialization)

The frontend project will be created in Phase 2 using Vite, React, and TypeScript according to the following modular folder architecture:

```
src/
├── app/                                    # Application entry, providers, global layout
│   ├── App.tsx
│   ├── Providers.tsx                       # React Query, AuthProvider, I18nProvider
│   └── router.tsx                          # Route definitions (Customer & Admin trees)
├── components/                             # Reusable UI component library
│   └── ui/                                 # Button, Input, Modal, Badge, Card, Select
├── features/                               # Domain-driven feature modules
│   ├── catalog/                            # Public product browsing, search, filters
│   │   ├── components/                     # ProductCard, PackSelector, StockBadge
│   │   ├── hooks/                          # useProducts, useProductSearch
│   │   └── services/                       # catalogService.ts
│   ├── cart/                               # Shopping cart management
│   │   ├── components/                     # CartDrawer, CartItem, DeliveryToggle
│   │   ├── hooks/                          # useCart, useCartValidation
│   │   └── store/                          # cartStore (localStorage sync)
│   ├── bookings/                           # Customer booking submission & tracking
│   │   ├── components/                     # BookingForm, WhatsAppShare, BookingStatus
│   │   ├── hooks/                          # useCreateBooking, useBookingDetails
│   │   └── services/                       # bookingService.ts
│   ├── account/                            # Customer profile, language preferences
│   │   ├── components/                     # ProfileView, OrderHistory
│   │   └── hooks/                          # useCustomerProfile
│   └── admin/                              # Internal store management domain
│       ├── dashboard/                      # KPI summary, quick actions, low stock alerts
│       ├── products/                       # Product CRUD, pack size setup, MRP config
│       ├── categories/                     # Category & target pest tag management
│       ├── inventory/                      # Stock view, manual adjustments, audit history
│       ├── suppliers/                      # Supplier master directory (Admin only)
│       ├── purchases/                      # Goods receipt, batch/expiry logging, WAC
│       ├── bookings/                       # Booking review, fulfillment, status changes
│       ├── sales/                          # Counter billing & checkout, thermal receipt print
│       ├── deliveries/                     # Local delivery assignment & tracking
│       ├── customers/                      # Customer list, credit limits, udhaar history
│       ├── reports/                        # Daily sales, profit margin, inventory value
│       └── settings/                       # Shop details, pesticide license details, UPI QR config
├── lib/                                    # Core infrastructure libraries
│   ├── supabase.ts                         # Supabase client initialization
│   ├── api.ts                              # PostgREST and RPC client abstractions
│   ├── utils.ts                            # Class merge, currency & date formatting
│   └── constants.ts                        # System constants, status enums
├── hooks/                                  # Global cross-cutting hooks
│   ├── useAuth.ts                          # Current user, session, role claims
│   ├── useNetworkStatus.ts                 # Online/offline detector for PWA
│   └── useDebounce.ts                      # Search query debouncing
├── routes/                                 # Route guard wrappers & error boundaries
│   ├── ProtectedRoute.tsx                  # Authenticated route guard
│   └── RoleGuard.tsx                       # Role-based route authorization
├── types/                                  # Global TypeScript interfaces & DB types
│   ├── database.types.ts                   # Supabase generated database definitions
│   ├── domain.types.ts                     # Business domain model interfaces
│   └── api.types.ts                        # RPC request & response payloads
└── i18n/                                   # Bilingual internationalization
    ├── config.ts                           # i18n initialization
    ├── en.json                             # English strings
    └── te.json                             # Telugu strings (తెలుగు అనువాదాలు)
```

---

## 5. Development vs. Production Isolation

To guarantee financial and inventory data integrity:
- **Development Environment**: Isolated Supabase project for automated migrations, test fixtures, and feature development.
- **Production Environment**: High-reliability Supabase Pro project hosted in the Mumbai/India AWS region (`ap-south-1`) for lowest latency, equipped with automated daily backups and Point-In-Time Recovery (PITR).
