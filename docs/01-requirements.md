# System Requirements Specification: Vighnesh AgroCare

## 1. Project Overview
**Vighnesh AgroCare** is a production Progressive Web App (PWA) tailored for an Indian retail pesticide shop (crop protection agrochemicals) operating in **Andhra Pradesh, India** (see Decision D18).

The system serves two primary stakeholder groups:
1. **Farmers & Agricultural Customers**: Fast, lightweight bilingual (English & Telugu) interface for browsing crop protection agrochemicals (insecticides, fungicides, herbicides, etc.), verifying retail prices (MRP and selling price) and availability, and booking products for shop pickup or local delivery.
2. **Shop Owner & Store Staff**: A robust administrative management dashboard for real-time inventory tracking, counter billing and sales, purchase goods receipt with batch and expiry tracking, supplier management, customer credit ledger (udhaar), local delivery coordination, and business reporting.

---

## 2. Functional Requirements by Module

### 2.1 Product Catalog
- **FR-CAT-01 Product Hierarchy & Categories**: Products must be categorized into crop protection segments: Insecticides, Fungicides, Herbicides, Acaricides, Nematicides, Rodenticides, and Bio-pesticides (configurable via a `categories` table in the database per Decision D19).
- **FR-CAT-02 Independent Pack Sizes**: Each distinct pack size (e.g. 50ml, 100ml, 250ml, 500ml, 1L, 5L, 100g, 250g, 500g, 1kg) must be modeled as a distinct sellable product record with its own SKU, MRP, selling price, and stock level. Example: *Imidacloprid 17.8% SL 100 ml* and *Imidacloprid 17.8% SL 250 ml* are separate product records.
- **FR-CAT-03 Regulatory & Statutory Product Fields**: Product records must support the following statutory and technical fields (Decision D20):
  - CIB&RC registration number (Central Insecticides Board & Registration Committee)
  - Manufacturer name
  - Technical name (active ingredient, e.g. *Imidacloprid 17.8% SL*, *Chlorpyrifos 20% EC*)
  - Formulation type (e.g. EC, WP, SC, SL, GR, WG, SP)
  - Toxicity colour class (Red: Extremely Toxic, Yellow: Highly Toxic, Blue: Moderately Toxic, Green: Slightly Toxic)
  - Batch number
  - Manufacturing date
  - Expiry date
  *Exact legally required fields TO BE VERIFIED WITH DEPARTMENT OF AGRICULTURE, ANDHRA PRADESH.*
- **FR-CAT-04 Safety, Antidote & Label Information**: Safety and label instructions (including antidotes, precautions, pre-harvest intervals / waiting periods, target pests, and dosage recommendations) must be entered directly by the shop staff and displayed strictly as entered. The application must **never generate automated claims or AI assertions** regarding product effectiveness, chemical safety, or crop suitability.
- **FR-CAT-05 Bilingual Information**: Product cards and detail views must display English trade names, technical chemical active ingredients, and localized Telugu names and usage instructions entered by the shop.
- **FR-CAT-06 Pricing Visibility**: Display Maximum Retail Price (MRP), actual selling price (tax-inclusive), and savings/discount amount clearly. Selling price cannot exceed MRP.
- **FR-CAT-07 Stock Availability Badges**: Display stock availability indicators (`In Stock`, `Low Stock`, `Out of Stock`) computed strictly from `available_stock` (`physical_stock - reserved_stock`).
- **FR-CAT-08 Operational Restriction Badges**: Products marked with `allow_online_booking = false` must display as "In-Store Only". Products with `requires_manual_review = true` must inform customers that booking requires staff review and confirmation before pickup.
- **FR-CAT-09 Search & Filtering**: Fast full-text search across brand name, technical active ingredient, target crop, target pest/weed, toxicity class, and category.

### 2.2 Shopping Cart
- **FR-CRT-01 Client-Side Cart**: Cart must persist across browser refreshes via localStorage while maintaining user isolation upon login/logout.
- **FR-CRT-02 Real-Time Stock Validation**: Cart items must be dynamically checked against currently available stock (`available_stock`). Users cannot add or increment items beyond available stock.
- **FR-CRT-03 Pack Size Selection**: Customers must be able to switch between available pack sizes directly from the product view or cart.
- **FR-CRT-04 Delivery Method Choice**: At cart checkout, customer selects between **Store Pickup** and **Home/Village Delivery**. Note: `allow_home_delivery` defaults to `false` for every product due to transport safety restrictions on agrochemicals; delivery is only selectable if all cart items allow delivery.
- **FR-CRT-05 Subtotal Calculation**: Dynamic computation of item totals, tax-inclusive subtotal, and estimated delivery fee (if applicable).

### 2.3 Booking & Concurrency-Safe Reservations
- **FR-BKG-01 Booking Creation**: The customer submits a booking containing one or more products. The backend executes a database transaction (`create_booking`) that locks stock using `SELECT ... FOR UPDATE` and records a reservation.
- **FR-BKG-02 24-Hour Stock Reservation**: Created bookings hold stock in `reserved_stock` for exactly 24 hours.
- **FR-BKG-03 Reservation Expiry**: An automated background job scans unfulfilled bookings older than 24 hours and transitions them to `cancelled`, automatically releasing the reserved stock back to `available_stock`.
- **FR-BKG-04 Idempotency Protection**: Every booking submission requires an idempotency key generated by the client to prevent duplicate bookings during network retries or double clicks.
- **FR-BKG-05 Booking Status Lifecycle**:
  - `pending`: Booking placed, awaiting customer arrival or staff confirmation.
  - `confirmed`: Staff verified stock availability and set items aside.
  - `ready_for_pickup`: Packed and waiting at the counter.
  - `out_for_delivery`: Assigned to delivery agent.
  - `completed`: Goods handed over and payment received/settled.
  - `cancelled`: Cancelled by user, rejected by admin, or auto-expired after 24h.
- **FR-BKG-06 WhatsApp Handoff**: Upon booking confirmation, provide a single-tap WhatsApp link (`wa.me`) pre-populated with order number, product items, and pickup/delivery details.

### 2.4 Inventory Management
- **FR-INV-01 Tri-Partite Stock Accounting**:
  - `physical_stock`: Total actual inventory physically present inside the retail shop.
  - `reserved_stock`: Inventory committed to active, unexpired customer bookings.
  - `available_stock`: Computable as `physical_stock - reserved_stock`. Must never drop below zero.
- **FR-INV-02 Append-Only Ledger**: All stock changes must be recorded in an immutable `inventory_movements` table. Direct editing or deletion of stock ledger rows is strictly prohibited.
- **FR-INV-03 Movement Types**:
  - `purchase_receipt` (+ physical)
  - `sale` (- physical; also - reserved if fulfilling booking)
  - `reservation` (+ reserved)
  - `reservation_release` (- reserved)
  - `adjustment_in` (+ physical)
  - `adjustment_out` (- physical)
  - `damage` (- physical)
  - `expiry` (- physical)
  - `customer_return` (+ physical)
  - `supplier_return` (- physical)
- **FR-INV-04 Low Stock Alerts**: Configurable low-stock threshold per product with automated visual indicators on the admin dashboard.
- **FR-INV-05 Audit & Stock Adjustments**: Stock reconciliation screen allowing authorized admins to record physical audit variances (`adjustment_in`, `adjustment_out`, `damage`, `expiry`) with mandatory audit notes.

### 2.5 Supplier Management
- **FR-SUP-01 Supplier Directory**: CRUD management of licensed agrochemical and pesticide manufacturers and authorized distributors.
- **FR-SUP-02 Supplier Metadata**: Company legal name, trade name, GSTIN, registered address, primary contact person, phone number, email, and credit terms (days).
- **FR-SUP-03 Confidentiality Protection**: Supplier details and contact information must be strictly visible only to users with the `admin` role.

### 2.6 Purchases & Goods Receipt
- **FR-PUR-01 Goods Receipt Recording**: Admins record received consignments against supplier invoices (`receive_purchase` function).
- **FR-PUR-02 Batch & Expiry Capture**: Every received line item must capture manufacturer batch number and expiry date for statutory compliance under the Insecticides Rules.
- **FR-PUR-03 Automatic WAC Calculation**: Ingestion of received items automatically recomputes the product's Weighted Average Cost (WAC) based on existing physical quantity and new purchase cost.
- **FR-PUR-04 Purchase Costs Confidentiality**: Purchase cost price per unit and supplier invoice amounts must never be exposed to `employee` or `delivery` roles.

### 2.7 Sales & Counter Billing
- **FR-SAL-01 Counter Sales**: Fast counter billing screen allowing staff to scan/search items, select pack sizes, and complete immediate sales.
- **FR-SAL-02 Booking Fulfillment**: Fulfill online bookings at the counter by looking up booking ID or customer phone number, converting reservation into a finalized sale.
- **FR-SAL-03 MRP Enforcement**: Unit sale price is strictly validated to ensure `sale_price <= mrp`. Any attempt to bill higher than MRP is rejected at the database level.
- **FR-SAL-04 Tax-Inclusive Computation**: Line items compute base taxable amount and GST breakdown from the tax-inclusive retail price.
- **FR-SAL-05 Printable Thermal Receipts**: Formatted 3-inch (80mm) thermal receipt output and digital shareable receipt format. Actual statutory format TO BE VERIFIED WITH ACCOUNTANT.

### 2.8 Local Delivery Management
- **FR-DEL-01 Order Assignment**: Admin assigns local delivery bookings to users with the `delivery` role.
- **FR-DEL-02 Delivery Agent Interface**: Mobile-optimized view for delivery agents displaying customer name, contact phone, destination village/address, items to deliver, and payment collection requirement.
- **FR-DEL-03 Delivery Status Updates**: Delivery agent marks orders as `out_for_delivery`, `delivered`, or `delivery_failed` with failure reason.

### 2.9 Payments Management
- **FR-PAY-01 Supported Payment Modes**: Cash, UPI (Dynamic/Static QR), and Customer Credit (Udhaar).
- **FR-PAY-02 Split Payments**: Support split tender for counter sales (e.g. part cash, part UPI, part credit).
- **FR-PAY-03 Payment Recording**: Atomic logging of payments in `sale_payments` linked to sales invoices.

### 2.10 Customer Credit Ledger (Udhaar Khata)
- **FR-CRD-01 Credit Account Creation**: Admin can enable a credit line for a registered customer and set a maximum `credit_limit`.
- **FR-CRD-02 Credit Limit Enforcement**: A sale cannot be booked on credit if the transaction would cause the customer's outstanding balance to exceed their assigned credit limit.
- **FR-CRD-03 Append-Only Credit Ledger**: All credit additions (sales on credit) and credit repayments (cash/UPI payments received from customer) recorded in `customer_credit_ledger`.
- **FR-CRD-04 Customer Statement**: Printable and WhatsApp-shareable ledger statement showing transaction dates, invoice references, repayments, and running outstanding balance.

### 2.11 Reports & Financial Analytics
- **FR-REP-01 Daily Sales Summary**: Daily gross sales, payment mode breakdown (Cash, UPI, Credit), and sales volume by pesticide category.
- **FR-REP-02 Inventory Valuation**: Total valuation of current physical inventory calculated using Weighted Average Cost (WAC). Restricted to `admin`.
- **FR-REP-03 Profit & Margin Reports**: Gross profit report computed as `(Selling Price - Weighted Average Cost) * Quantity Sold`. Strictly restricted to `admin`.
- **FR-REP-04 Outstanding Credit (Udhaar) Report**: Aging list of customer outstanding credit balances and collection tracking.
- **FR-REP-05 Fast & Slow Moving Stock**: Analytical breakdown of inventory velocity to prevent chemical expiry losses.

### 2.12 Authentication & Access Control
- **FR-AUT-01 Customer Authentication**: Email and password registration and login via Supabase Auth.
- **FR-AUT-02 Guest Booking with Rate Limiting**: Unauthenticated customers can create bookings by providing Name, Phone Number, and Village/Address, protected against bot abuse by CAPTCHA verification (Cloudflare Turnstile).
- **FR-AUT-03 Staff & Role Assignment**: Internal user accounts (`admin`, `employee`, `delivery`) cannot self-register; roles are assigned exclusively by an existing administrator.
- **FR-AUT-04 Session Security**: Secure JWT session handling, automatic token refresh, and instantaneous revocation on logout.

### 2.13 Progressive Web App (PWA)
- **FR-PWA-01 Installability**: Web App Manifest supporting "Add to Home Screen" on Android (Chrome) and iOS (Safari).
- **FR-PWA-02 Offline Shell & Static Caching**: Service Worker caching of HTML application shell, scripts, styles, icons, and fonts for immediate load in low-connectivity rural environments.
- **FR-PWA-03 Offline Data Policy**: Static assets and public catalog browsing shell cached; authenticated API data and financial transactions NEVER cached in insecure browser storage.

### 2.14 Internationalization (i18n) & Localization
- **FR-I18N-01 Languages**: Full interface translation in English and Telugu (తెలుగు).
- **FR-I18N-02 Language Switcher**: Persistent language selection stored in client preferences with seamless instant toggle without page reload.
- **FR-I18N-03 Typography**: Inclusion of high-legibility Telugu fonts (e.g. Noto Sans Telugu) ensuring correct rendering of conjunct consonants (vattulu and guninthalu).
- **FR-I18N-04 Localized Formats**: Indian numbering format (e.g. ₹1,25,000.00) and Indian standard date formats (DD/MM/YYYY).

---

## 3. Non-Functional Requirements

### 3.1 Concurrency & Data Consistency
- **NFR-CON-01 Zero Overselling**: Concurrency control via PostgreSQL row-level locks (`SELECT ... FOR UPDATE`) inside stored functions to prevent race conditions during simultaneous bookings of scarce seasonal pesticide stock.
- **NFR-CON-02 Zero Negative Stock**: Database constraints (`physical_stock >= 0`, `available_stock >= 0`) guarantee stock balances can never enter negative values under any circumstance.
- **NFR-CON-03 Idempotent Transactions**: All stock alteration, booking creation, and payment operations must support idempotency keys to eliminate duplicate records caused by client retries or network drops.

### 3.2 Performance & Responsiveness
- **NFR-PRF-01 Low Bandwidth Optimization**: First Contentful Paint (FCP) under 1.8 seconds and Time to Interactive (TTI) under 3.0 seconds on simulated 3G mobile networks.
- **NFR-PRF-02 Asset Optimization**: Total initial JavaScript bundle size kept under 250 KB (compressed); responsive image delivery for product photography.
- **NFR-PRF-03 Database Latency**: PostgREST / RPC function execution time under 150ms for 95th percentile requests.

### 3.3 Security & Authorization
- **NFR-SEC-01 Row Level Security (RLS)**: Mandatory RLS enabled on 100% of public schema tables with default deny policy.
- **NFR-SEC-02 Principle of Least Privilege**: Counter employees and delivery staff have zero access to purchase prices, supplier records, margins, or owner financial metrics.
- **NFR-SEC-03 Safe Stored Procedures**: All PostgreSQL stored procedures defined with `SECURITY DEFINER` must explicitly set `search_path = public, pg_temp;` and validate caller identity and roles internally.
- **NFR-SEC-04 Zero Secret Leakage**: Service role keys, database passwords, and third-party API secrets must never be embedded in client-side code, frontend environment files, or client repository commits.

### 3.4 Regulatory & Compliance Hooks
- **NFR-REG-01 Statutory Records**: Data schema must accommodate mandatory retail register and inspection requirements under the Insecticides Act, 1968 and Insecticides Rules, 1971. TO BE VERIFIED WITH DEPARTMENT OF AGRICULTURE, ANDHRA PRADESH.
- **NFR-REG-02 Tax Compliance**: Invoicing data structure must comply with Andhra Pradesh GST retail invoice guidelines for crop protection chemicals (typically 18% GST). TO BE VERIFIED WITH ACCOUNTANT.
- **NFR-REG-03 Legal Metrology**: Selling price enforcement ensuring strict compliance with Legal Metrology (Packaged Commodities) Rules, barring sales above printed MRP.
