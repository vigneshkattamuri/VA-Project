# Implementation Roadmap: Vighnesh AgroCare

This roadmap outlines the 20 structured implementation phases for **Vighnesh AgroCare**. Each phase defines an explicit Goal and unambiguous Acceptance Criteria.

---

### Phase 1: Requirements, Architecture & Structural Foundation
- **Goal**: Establish core documentation, architectural decisions, inventory rules, security specifications, legal verification checklist, and Git repository structure without writing application code.
- **Acceptance Criteria**:
  - [x] Dedicated branch `phase-1-foundation` active.
  - [x] Comprehensive `.gitignore` and sanitized `.env.example` created.
  - [x] Complete specifications in `docs/` (`00-decisions-log.md` through `07-legal-checklist.md` and `adr/0001-architecture-decisions.md`).
  - [x] Folder skeleton created with `.gitkeep` files.
  - [x] Zero secrets, zero premature dependencies, and zero application code committed.

---

### Phase 2: React + Vite + TypeScript + Tailwind CSS + i18n Setup
- **Goal**: Initialize clean frontend tooling with Vite, TypeScript, Tailwind CSS, routing skeleton, and bilingual i18n framework (English/Telugu).
- **Acceptance Criteria**:
  - [ ] Vite React TypeScript application initialized in non-interactive mode.
  - [ ] Tailwind CSS configured with a tailored, modern agricultural color palette.
  - [ ] Telugu-capable typography (e.g. Noto Sans Telugu) loaded and configured.
  - [ ] `i18next` initialized with English and Telugu key dictionaries and persistent language switcher.
  - [ ] Planned directory structure under `src/` created.

---

### Phase 3: Branding, Core UI Components & Dual Layouts
- **Goal**: Implement high-quality visual identity and reusable UI component primitives with distinct Customer (`/`) and Admin (`/admin`) layouts.
- **Acceptance Criteria**:
  - [ ] Design system tokens and UI components (`Button`, `Input`, `Select`, `Modal`, `Badge`, `Card`, `Table`).
  - [ ] Responsive navigation shell for farmers (mobile-first bottom navigation / header).
  - [ ] Sidebar navigation layout for the admin management portal.
  - [ ] Accessible color contrasts and touch target sizes.

---

### Phase 4: Database Schema & Supabase Migrations
- **Goal**: Author reproducible SQL migrations defining tables, relations, constraints, and audit columns for retail agrochemicals (pesticides, PGRs, micronutrients).
- **Acceptance Criteria**:
  - [ ] Migrations for `products` supporting statutory fields per D20: registration/licence number (CIB&RC or applicable), manufacturer, technical name/composition, formulation, toxicity colour class (where applicable), batch number, manufacturing date, and expiry date.
  - [ ] Revisit batch-level stock model per Decision D02 (Provisional).
  - [ ] Tables for `product_stock`, `inventory_movements`, `suppliers`, `purchases` (with draft status), `sales`, `bookings`, `customer_credit_ledger`, and `shop_settings`.
  - [ ] Constraints enforced (`selling_price <= mrp`, `physical_stock >= 0`, `reserved_stock >= 0`).
  - [ ] Append-only movement types defined, with mandatory reason note requirement for stock adjustments.
  - [ ] Migration applied cleanly to local/development Supabase instance.

---

### Phase 5: Authentication, User Roles & Row Level Security (RLS)
- **Goal**: Implement authentication flows and secure 100% of tables with PostgreSQL Row Level Security.
- **Acceptance Criteria**:
  - [ ] Supabase Auth integrated for customer login/registration.
  - [ ] Separate `user_roles` table with `admin`, `employee`, `delivery`, `customer` assignments.
  - [ ] RLS policies enabled on all tables; default deny verified.
  - [ ] Employees blocked from viewing purchase costs, supplier records, and profit reports.
  - [ ] Customers restricted strictly to their own data per D21 (no access to raw stock counts, costs, or other customers' data).

---

### Phase 6: Customer Home & Product Catalog
- **Goal**: Deliver a high-performance, mobile-first crop protection and nutrition browsing experience in English and Telugu.
- **Acceptance Criteria**:
  - [ ] Categorized browsing across 9 initial database-configurable categories (per D19): Insecticides, Fungicides, Herbicides, Acaricides, Nematicides, Rodenticides, Bio-pesticides, Plant Growth Regulators, Micronutrients.
  - [ ] Customer-facing data boundary strictly enforced per D21: product name, price, images, product details, availability status badge (`In Stock`, `Low Stock`, `Out of Stock`), booking option, accepted payment options, and own booking history. Never expose raw numerical stock counts, supplier costs, or other customers' data.
  - [ ] Pack size selector per product card with dynamic MRP and selling price display (`selling_price <= mrp`).
  - [ ] Search and filter by brand, category, formulation, and toxicity class.
  - [ ] Safety and label information displayed strictly as entered by the shop, with no system-generated claims about efficacy, safety, or crop suitability.

---

### Phase 7: Admin Product & Category Management
- **Goal**: Build management interfaces for creating, editing, and categorizing sellable agrochemical products and pack sizes.
- **Acceptance Criteria**:
  - [ ] Admin forms to create and edit products with independent pack sizes and statutory fields per D20 (registration/licence no., manufacturer, composition, formulation, toxicity colour class, batch no., mfg date, expiry date).
  - [ ] Product-level flags: `allow_online_booking`, `allow_home_delivery` (defaults to `false` for every product per D15), `requires_manual_review`.
  - [ ] MRP and selling price validation rules (`selling_price <= mrp`).
  - [ ] Category and target pest/application tag management screens.

---

### Phase 8: Inventory & Stock Movements
- **Goal**: Deliver store inventory overview, stock adjustment interface, and audit trail ledger.
- **Acceptance Criteria**:
  - [ ] Real-time inventory table showing physical, reserved, and available stock.
  - [ ] Stock adjustment modal triggering `adjust_stock` function (`adjustment_in`, `adjustment_out`, `damage`, `expiry`) with mandatory audit reason note.
  - [ ] Filterable audit history view of the immutable `inventory_movements` ledger.
  - [ ] Automated visual low-stock warnings.

---

### Phase 9: Supplier Management
- **Goal**: Create protected supplier master directory for store administration.
- **Acceptance Criteria**:
  - [ ] CRUD operations for licensed agrochemical manufacturers and authorized distributors.
  - [ ] Storage of legal trade name, GSTIN, primary contacts, and credit terms.
  - [ ] Strict RLS enforcement ensuring supplier data is inaccessible to `employee` and `delivery` roles.

---

### Phase 10: Purchases & Goods Receipt (Stock Upload Workflow)
- **Goal**: Implement stock upload workflow: draft purchase orders, goods receipt with batch/expiry logging, adjustments with reason, and automated WAC recalculation.
- **Acceptance Criteria**:
  - [ ] Draft purchase order management: create, review, update, or discard draft purchase bills without affecting live inventory.
  - [ ] Goods receipt screen against draft purchases: mandatory capture of batch number, manufacturing date, and expiry date per line item (per D20).
  - [ ] Backend execution of atomic `receive_purchase` function updating physical stock, appending audit ledger movements, and recalculating Weighted Average Cost.
  - [ ] Purchase history and invoice archives (Admin only).

---

### Phase 11: Shopping Cart & Customer Checkout Flow
- **Goal**: Build a persistent client-side cart and checkout flow supporting store pickup, conditional local delivery, accepted payment disclosures, and statutory privacy notices.
- **Acceptance Criteria**:
  - [ ] Cart drawer/view with live stock checks against `available_stock`.
  - [ ] Selection between Store Pickup and Home/Village Delivery (conditioned on `allow_home_delivery`).
  - [ ] Guest checkout flow requiring Customer Name and Mobile Number with rate limiting.
  - [ ] Statutory privacy notice displayed prominently at booking for collected customer name and mobile number under the Digital Personal Data Protection Act, 2023.
  - [ ] Accepted payment options displayed clearly per D23 (Pay at Shop, Cash on Delivery; online gateway deferred).
  - [ ] Clear order summary displaying tax-inclusive pricing.

---

### Phase 12: Booking Engine & Concurrency-Safe Reservations
- **Goal**: Implement transactional booking creation with 24-hour stock holds and concurrency locks.
- **Acceptance Criteria**:
  - [ ] `create_booking` RPC utilizing `SELECT ... FOR UPDATE` row locks.
  - [ ] Idempotency key validation preventing duplicate submissions.
  - [ ] 24-hour reservation hold placed on `reserved_stock`.
  - [ ] Scheduled cron function `expire_stale_bookings` releasing expired holds.
  - [ ] Direct WhatsApp handoff link (`wa.me`) generated on booking completion screen.

---

### Phase 13: Admin Booking Management & Fulfillment
- **Goal**: Build store counter booking management dashboard for review, packing, and fulfillment.
- **Acceptance Criteria**:
  - [ ] Real-time list of customer bookings filtered by status (`pending`, `confirmed`, `ready_for_pickup`).
  - [ ] Staff action to confirm booking or reject/cancel with stock release.
  - [ ] Customer search by phone number or booking reference code.
  - [ ] Manual review verification workflow for flagged chemical items.

---

### Phase 14: Counter Sales, Invoicing & Payments
- **Goal**: Build rapid counter sales billing interface with customer name/mobile capture, payment recording, and thermal receipt generation.
- **Acceptance Criteria**:
  - [ ] Fast item lookup and SKU selection for walk-in counter sales.
  - [ ] Billing workflow capturing Customer Name and Mobile Number on sales receipts.
  - [ ] Admin records payment method (Cash, UPI QR, Split Tender) and status per D23.
  - [ ] Conversion of online bookings into finalized sales.
  - [ ] Atomic `complete_sale` execution validating `selling_price <= mrp`.
  - [ ] Formatted 3-inch (80mm) thermal receipt view and printable bill.

---

### Phase 15: Local Delivery Management
- **Goal**: Dispatch, assign, and track local home/village deliveries.
- **Acceptance Criteria**:
  - [ ] Admin assignment of orders to delivery agents.
  - [ ] Mobile-optimized Delivery Agent portal displaying active routes, customer contact, and delivery notes.
  - [ ] Delivery status updates (`out_for_delivery`, `delivered`, `delivery_failed`).
  - [ ] Collection confirmation for Cash-on-Delivery (COD) or UPI on Delivery.

---

### Phase 16: Admin Business Reports & Analytics
- **Goal**: Provide admin-only business intelligence, customer purchase history, inventory valuation, and CSV export per D22.
- **Acceptance Criteria**:
  - [ ] Admin customer purchase view: search customer purchase history (customer name, mobile number, purchased items, total bill, date).
  - [ ] Best sellers and sales volume analytics per category/product.
  - [ ] Remaining stock, low stock alert list, and batch expiry tracking report.
  - [ ] Financial reports: daily sales, purchase totals, and gross profit `(Selling Price - WAC) * Qty` strictly protected for `admin`.
  - [ ] CSV export functionality for all sales, purchases, customer history, and stock reports.

---

### Phase 17: Progressive Web App (PWA) Offline & Installability
- **Goal**: Enable reliable PWA installation and offline static shell caching.
- **Acceptance Criteria**:
  - [ ] Web App Manifest with icons, theme colors, and standalone display mode.
  - [ ] Service worker caching static assets, fonts, and catalog shell.
  - [ ] Zero caching of authenticated customer or financial API responses.
  - [ ] Offline notification banner when network connection drops.

---

### Phase 18: Testing & Security Hardening
- **Goal**: Execute comprehensive unit, integration, and end-to-end tests across all edge cases.
- **Acceptance Criteria**:
  - [ ] Vitest unit tests verifying WAC formulas, tax math, and date helpers.
  - [ ] Supabase SQL tests validating PL/pgSQL functions and RLS boundary isolation.
  - [ ] Playwright E2E tests validating concurrent bookings, checkout, and admin counter billing.
  - [ ] Automated security audit confirming zero public exposure of service role keys or supplier costs.

---

### Phase 19: Production Deployment & Infrastructure
- **Goal**: Deploy production infrastructure on Supabase Pro and Vercel CDN.
- **Acceptance Criteria**:
  - [ ] Production Supabase instance configured with automated daily backups and PITR.
  - [ ] Database migrations applied cleanly to production.
  - [ ] Vercel production deployment configured with custom domain and SSL.
  - [ ] Environment variables verified and isolated from staging.

---

### Phase 20: Final Handover, Training & Documentation
- **Goal**: Deliver owner operational manual, staff user guides, and technical handover documentation.
- **Acceptance Criteria**:
  - [ ] Bilingual owner & counter staff user guide (English & Telugu).
  - [ ] Operational troubleshooting runbook for store owners.
  - [ ] Final architecture and disaster recovery documentation archived.
