# System Architecture: Vighnesh AgroCare

## 1. Architectural Strategy

**Vighnesh AgroCare** is structured as a high-performance single-page Progressive Web Application (PWA) with a headless, database-centric backend powered by Supabase and PostgreSQL, tailored specifically for an agrochemical retail shop (crop-protection and crop-nutrition products: pesticides, plant growth regulators, and micronutrients) in Andhra Pradesh, India.

### Core Architectural Principles
1. **Single Frontend Application with Segmented Layouts**: A unified React + TypeScript codebase serves both public customers (farmers) and internal store operators via distinct route hierarchies and layouts:
   - Customer Portal (`/`): Lightweight, mobile-first, bilingual (English/Telugu), optimized for fast browsing of crop-protection and crop-nutrition products, cart management, privacy-compliant booking, and accepted payment options.
   - Admin & Staff Portal (`/admin`): Desktop/tablet-optimized, counter sales and billing (capturing customer name and mobile), inventory audit, stock upload workflows (draft purchases, goods receipt), delivery management, and operational analytics.
2. **Database-Centric Business Logic (PL/pgSQL Functions)**: All critical business transactions—such as stock reservations, booking transitions, purchase receipts, inventory adjustments, counter sales billing, and payment processing—execute exclusively inside PostgreSQL stored functions. The frontend browser **never** writes directly to inventory or financial tables.
3. **Data Boundary & Protection (Decision D21)**: Customer-facing endpoints strictly restrict data to product name, price, images, technical details, and general availability status (`In Stock`, `Low Stock`, `Out of Stock`). Exact stock numbers, wholesale purchase costs, supplier records, and other customers' purchase histories are strictly guarded by PostgreSQL Row Level Security (RLS).
4. **Concurrency Safety & ACID Guarantees**: Stock deduction and booking reservation utilize explicit row locking (`SELECT ... FOR UPDATE`), transaction isolation, and database-level idempotency checks.
5. **Minimal Edge Functions**: Supabase Edge Functions (TypeScript/Deno) are utilized solely when server-side secrets or third-party webhooks are involved (e.g. CAPTCHA verification, future SMS gateway integration). Public and authenticated database interactions occur via PostgREST and RPC calls governed by strict Row Level Security (RLS).
6. **Strict Environment Isolation**: Independent development and production Supabase projects to safeguard financial and inventory integrity.

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
|   |   - Agrochemical Catalog & Search     |   |   - Counter Sales & Billing           | |
|   |   - Cart & Checkout (Privacy Notice)  |   |   - Stock Upload & Goods Receipt      | |
|   |   - 24h Booking Creation              |   |   - Inventory Adjustments with Reason | |
|   |   - Payment Options Display           |   |   - Customer Purchase Analytics       | |
|   |   - WhatsApp Integration (wa.me)      |   |   - Udhaar Credit Ledger              | |
|   |   - Customer Own Booking History      |   |   - Best Sellers & Expiry Reports     | |
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
|   |   |   |  - Customer-facing data restriction rule (D21) enforced         |   |   |   |
|   |   |   +-----------------------------------------------------------------+   |   |   |
|   |   |                                 |                                       |   |   |
|   |   |   +-----------------------------v-----------------------------------+   |   |   |
|   |   |   |       Core Business Logic Engine (PL/pgSQL Stored RPCs)         |   |   |   |
|   |   |   |                                                                 |   |   |   |
|   |   |   |   * create_booking(items, customer_info, idempotency_key)       |   |   |   |
|   |   |   |   * transition_booking(booking_id, new_status)                  |   |   |   |
|   |   |   |   * cancel_booking(booking_id, reason)                          |   |   |   |
|   |   |   |   * receive_purchase(supplier_id, invoice_meta, items)          |   |   |   |
|   |   |   |   * adjust_stock(product_id, qty, movement_type, reason_notes)  |   |   |   |
|   |   |   |   * complete_sale(items, payments, customer_name, mobile)       |   |   |   |
|   |   |   |   * record_payment(sale_id, customer_id, amount, mode)          |   |   |   |
|   |   |   |                                                                 |   |   |   |
|   |   |   |   [ACID Transactions | SELECT ... FOR UPDATE | Idempotency Keys]    |   |   |   |
|   |   |   +-----------------------------------------------------------------+   |   |   |
|   |   |                                 |                                       |   |   |
|   |   |   +-----------------------------v-----------------------------------+   |   |   |
|   |   |   |                     Data Persistence Layer                      |   |   |   |
|   |   |   |                                                                 |   |   |   |
|   |   |   |   - products (licence/reg, composition, formulation, toxicity)  |   |   |   |
|   |   |   |   - product_stock (physical, reserved, WAC cost)                |   |   |   |
|   |   |   |   - inventory_movements (Immutable append-only ledger)          |   |   |   |
|   |   |   |   - bookings & booking_items (24h reservations)                 |   |   |   |
|   |   |   |   - sales & sale_items & sale_payments (customer name/mobile)   |   |   |   |
|   |   |   |   - customer_credit_ledger (Udhaar balance & limits)            |   |   |   |
|   |   |   |   - suppliers & purchase_orders (drafts) & purchase_items       |   |   |   |
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

## 3. Database-Centric Business Functions & Workflows

The browser client is strictly restricted from performing direct SQL `INSERT`, `UPDATE`, or `DELETE` operations on core inventory, sales, and accounting tables. All state mutations occur via PostgreSQL functions executed with `SECURITY DEFINER` privileges, guaranteeing transaction boundaries and audit enforcement.

### 3.1 `create_booking`
- **Parameters**: `p_customer_id`, `p_customer_name`, `p_phone`, `p_delivery_type`, `p_delivery_address`, `p_items` (JSON array of `product_id`, `quantity`), `p_idempotency_key`.
- **Execution**: Validates available stock, obtains row locks via `SELECT ... FOR UPDATE`, increments `reserved_stock`, inserts `inventory_movements` (`reservation`), and sets 24-hour expiration.

### 3.2 `transition_booking`
- **Parameters**: `p_booking_id`, `p_new_status`, `p_notes`.
- **Execution**: Validates allowed status transitions according to the state machine (`pending` -> `confirmed` -> `ready_for_pickup` -> `out_for_delivery` -> `completed`).

### 3.3 `cancel_booking`
- **Parameters**: `p_booking_id`, `p_cancellation_reason`.
- **Execution**: Locks booking row, releases `reserved_stock`, inserts `reservation_release` movement, and marks booking as `cancelled`.

### 3.4 `receive_purchase` (Stock Upload Workflow)
- **Parameters**: `p_supplier_id`, `p_invoice_number`, `p_invoice_date`, `p_items` (JSON array of `product_id`, `quantity`, `cost_price_per_unit`, `batch_number`, `mfg_date`, `expiry_date`), `p_idempotency_key`.
- **Execution**:
  - Ingests draft purchase orders or incoming goods receipts.
  - Locks product stock rows `FOR UPDATE`.
  - Recalculates New Weighted Average Cost (WAC) for each product.
  - Increments physical stock and records batch number, manufacturing date, and expiry date per goods-receipt line (`purchase_items` batch record). Batch data belongs strictly to stock receipts, not products.
  - Inserts `purchase_receipt` movement into `inventory_movements`.

### 3.5 `adjust_stock` (Stock Adjustment with Mandatory Reason)
- **Parameters**: `p_product_id`, `p_quantity`, `p_movement_type` (`adjustment_in`, `adjustment_out`, `damage`, `expiry`), `p_reason_notes`.
- **Execution**:
  - Restricted strictly to `admin`.
  - Mandatory non-empty `p_reason_notes` required for audit tracking.
  - Locks product stock `FOR UPDATE`.
  - Adjusts `physical_stock` and logs `inventory_movements`.

### 3.6 `complete_sale` (Counter Billing with Customer Details)
- **Parameters**: `p_customer_name`, `p_customer_mobile`, `p_booking_id` (optional), `p_items` (JSON array of `product_id`, `quantity`, `unit_price`), `p_payments` (JSON array of `mode`, `amount`), `p_idempotency_key`.
- **Execution**:
  - Validates `unit_price <= mrp`.
  - Captures customer name and mobile for statutory compliance under Rule 15 of Insecticides Rules.
  - Decrements `physical_stock` (and `reserved_stock` if linked to an active booking).
  - Inserts `sale`, `sale_items`, `sale_payments`, and `inventory_movements` (`sale`).

### 3.7 `record_payment`
- **Parameters**: `p_customer_id`, `p_sale_id` (optional), `p_amount`, `p_payment_mode` (`cash`, `upi`), `p_reference_number`.
- **Execution**: Records debt repayments towards outstanding credit balances in `customer_credit_ledger`.

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
│   │   ├── components/                     # BookingForm, WhatsAppShare, BookingStatus, PrivacyNotice
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
│       ├── purchases/                      # Draft purchases, goods receipt, batch/expiry logging, WAC
│       ├── bookings/                       # Booking review, fulfillment, status changes
│       ├── sales/                          # Counter billing, customer name/mobile, thermal receipt print
│       ├── deliveries/                     # Local delivery assignment & tracking
│       ├── customers/                      # Customer list, credit limits, udhaar history, purchase history
│       ├── reports/                        # Best sellers, remaining stock, expiry alerts, profit, CSV export
│       └── settings/                       # Shop details, licence details, UPI QR config
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
