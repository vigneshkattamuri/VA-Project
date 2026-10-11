# Security Model & Access Control: Vighnesh AgroCare

## 1. Role-Based Access Control (RBAC) Overview

The system strictly enforces the principle of least privilege across all user tiers. There are four discrete system roles:
1. **`admin`**: Store owner / primary manager. Unrestricted administrative, financial, operational, and reporting authority.
2. **`employee`**: Shop counter sales assistant. Operates counter billing, searches catalog, views stock availability, fulfills customer bookings. Strictly prohibited from viewing purchase costs, supplier records, margins, customer-wide purchase analytics, and profit reports.
3. **`delivery`**: Field delivery agent. Dispatches and delivers local customer orders. Strictly limited to assigned delivery orders, customer address/contact, and marking delivery/payment status.
4. **`customer`**: Retail buyer / local farmer. Can browse public catalog, view availability status, manage personal cart, create bookings with privacy notice, view personal order history, and update personal profile.

---

## 2. Permissions Matrix

| Resource / Action | Customer | Employee | Delivery | Admin |
| :--- | :---: | :---: | :---: | :---: |
| **Product Catalog (Public Data per D21)** | Yes | Yes | Yes | Yes |
| **Product Master (Create / Edit / Archive)** | No | No | No | Yes |
| **Product Availability Status (`In Stock`/`Low`/`Out`)** | Read | Read | Read | Read |
| **Exact Stock Counts (`physical_stock`, `reserved_stock`)** | **No (Blocked)** | Read | Read | Read |
| **Inventory Movements Ledger (Read)** | No | No | No | Yes |
| **Stock Adjustments (`adjust_stock` with reason)** | No | No | No | Yes |
| **Draft Purchases & Goods Receipt** | No | **No (Blocked)** | No | Yes |
| **Suppliers Directory (Read / Write)** | No | **No (Blocked)** | No | Yes |
| **Product Purchase Costs / Supplier Invoices** | No | **No (Blocked)** | No | **Yes (Admin Only)** |
| **Counter Sales & Billing (Name + Mobile)** | No | Yes | No | Yes |
| **Customer Purchase History (Name, Mobile, Items, Bill)** | Own Only | No | No | **Yes (Admin Only)** |
| **Best Sellers & Remaining Stock Analytics** | No | No | No | **Yes (Admin Only)** |
| **Chemical Expiry & Low Stock Alerts** | No | Read | No | **Yes (Admin Only)** |
| **Customer Credit Ledger (Udhaar) (Read)** | Own Balance | Read Balance | No | Full Read |
| **Modify Customer Credit Limits** | No | No | No | Yes |
| **Online Bookings (Create with Privacy Notice)** | Yes (Own / Guest) | Yes (Walk-in) | No | Yes |
| **Online Bookings (Transition / Fulfill)** | Cancel Own | Yes | Assigned Only | Yes |
| **Delivery Orders (View & Dispatch)** | Own Orders | View All | Assigned Only | Full Control |
| **Financial, Profit & CSV Export Reports** | No | **No (Blocked)** | No | **Yes (Admin Only)** |
| **Shop Settings (Licences, Tax, UPI Config)** | Read Public Info | Read Public Info | Read Public Info | Full Control |
| **User Role Assignment** | No | No | No | **Yes (Admin Only)** |

---

## 3. Customer Data Access Restrictions (Decision D21)

To protect commercial store confidentiality and individual farmer privacy:
- **Public & Customer Visibility Boundary**:
  - Allowed: Product brand/trade name, retail price (selling price and printed MRP), product imagery, technical details (composition, formulation, shop-entered dosage/safety notes), availability status indicator (`In Stock`, `Low Stock`, `Out of Stock`), accepted payment methods ("Pay at Shop", "Cash on Delivery"), and the customer's own authenticated booking records.
  - Prohibited: Numeric physical or reserved stock counts, wholesale acquisition costs, supplier identities, margins, and purchase histories of any other customers.
- **Enforcement**: Public database views and PostgreSQL Row Level Security (RLS) policies omit sensitive columns from all public/anon queries.

---

## 4. Role Assignment Security

- **Self-Assignment Prohibition**: Users can **never** assign or modify roles for themselves.
- **Implementation**:
  - Roles are stored in a dedicated `user_roles` table (`user_id`, `role`, `created_at`, `assigned_by`) outside the editable `profiles` table.
  - The `user_roles` table is protected by PostgreSQL RLS with `INSERT`, `UPDATE`, and `DELETE` policies restricted strictly to authenticated users with the `admin` role.
  - User registration via Supabase Auth defaults automatically to the `customer` role via a database trigger.
  - Elevated roles (`employee`, `delivery`, `admin`) can only be provisioned through an explicit admin action or initial setup script during deployment.

---

## 5. PostgreSQL Row Level Security (RLS) Principles

1. **Default Deny Strategy**: Every single table created in the `public` schema has Row Level Security explicitly enabled (`ALTER TABLE ... ENABLE ROW LEVEL SECURITY;`). No query returns data unless an explicit policy permits it.
2. **Read-Only Inventory Exposure**:
   - The public/customer tier only reads sanitized availability indicators through an API view.
   - Direct `INSERT`, `UPDATE`, or `DELETE` on `inventory_movements`, `product_stock`, `sales`, and `bookings` is completely denied to all client roles. All mutations must occur via validated PostgreSQL stored functions.
3. **Data Isolation by Tenant / Customer**:
   - Customers can only query bookings, orders, and ledger entries matching their authenticated UID (`auth.uid() = user_id`).
   - Guest bookings are secured via non-guessable UUIDs and tokenized booking references.
4. **Role Verification Helper**:
   - A secure database function `auth.has_role(required_role text)` inspects the verified `user_roles` table or decoded JWT app metadata, cached within the session transaction.

---

## 6. SECURITY DEFINER Stored Procedure Rules

Because all core business logic is encapsulated in PostgreSQL stored functions executed with `SECURITY DEFINER` privileges, the following strict hardening guidelines are mandatory:

1. **Explicit Search Path Setting**: Every `SECURITY DEFINER` function **must** include:
   ```sql
   SET search_path = public, pg_temp;
   ```
   This eliminates search_path hijacking attacks by malicious schema injection.
2. **Internal Caller Authentication & Authorization**: The function must actively verify the caller's identity and role at the top of the function body:
   ```sql
   IF auth.uid() IS NULL THEN
     RAISE EXCEPTION 'Authentication required.' USING ERRCODE = '42501';
   END IF;

   IF NOT auth.has_role('admin') THEN
     RAISE EXCEPTION 'Access denied: Administrator role required.' USING ERRCODE = '42501';
   END IF;
   ```
3. **Strict Parameter Validation**: Input JSON, numeric quantities, and text strings must be sanity-checked against boundary conditions (e.g. `quantity > 0`, `selling_price <= mrp`).

---

## 7. Personally Identifiable Information (PII) Protection & Privacy Notice

Agrochemical retail involves customer personal data regulated under the **Digital Personal Data Protection Act, 2023**:

### Identified PII Data Elements
- Customer Full Name
- Primary Mobile Phone Number
- Village, Mandal, and District Details
- Street / Farm Delivery Address
- Purchase & Consumption History (crop types and chemical usage)
- Credit Ledger Balances (Udhaar financial standing)

### Safeguards & Statutory Privacy Notice
- **Statutory Privacy Notice at Booking**: Whenever customer name and mobile number are collected (guest booking or registered checkout), a clear privacy notice explains that data is collected solely for order reservation, counter billing, and local delivery fulfillment.
- **Operational Need Isolation**: PII is restricted to store staff with an active operational need.
- **Delivery Agent Data Masking**: Delivery agents only receive customer name, phone, and delivery address for active dispatches; once delivered, historical contact details are masked.
- **Financial Privacy**: Customer credit balances are considered sensitive financial data and are hidden from public and delivery views.
- **Zero Log Leakage**: No PII is logged in application error trackers or client console outputs.

---

## 8. Progressive Web App (PWA) Caching Rules

To ensure client data privacy and prevent sensitive leakage on shared or public mobile devices:

1. **Zero Caching of Authenticated API Responses**:
   - The Service Worker cache (`CacheStorage`) must **never** store authenticated API requests, user profiles, order histories, booking records, or payment data.
   - Any API response from `/rest/v1/*` or `/rpc/*` must be fetched exclusively over network or served from memory, never persisted in Service Worker disk caches.
2. **Permitted Cache Targets**:
   - Application shell (HTML, bundled JavaScript, CSS).
   - Web fonts (including Telugu font files).
   - Static branding assets, icons, and placeholder images.
   - Public product catalog images from Supabase Storage (served with cache-control headers).
3. **Logout Cache Eviction**:
   - On user logout, all client-side state stored in `localStorage`, `sessionStorage`, and client query memory caches is purged immediately.
