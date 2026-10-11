# Security Model & Access Control: Vighnesh AgroCare

## 1. Role-Based Access Control (RBAC) Overview

The system strictly enforces the principle of least privilege across all user tiers. There are four discrete system roles:
1. **`admin`**: Store owner / primary manager. Unrestricted administrative and financial authority.
2. **`employee`**: Shop counter sales assistant. Operates counter billing, searches catalog, views stock availability, fulfills customer bookings. Prohibited from viewing purchase costs, supplier records, margins, and profit analytics.
3. **`delivery`**: Field delivery agent. Dispatches and delivers local customer orders. Strictly limited to assigned delivery orders, customer address/contact, and marking delivery/payment status.
4. **`customer`**: Retail buyer / local farmer. Can browse public catalog, manage personal cart, create bookings, view personal order history, and update personal profile.

---

## 2. Permissions Matrix

| Resource / Action | Customer | Employee | Delivery | Admin |
| :--- | :---: | :---: | :---: | :---: |
| **Product Catalog (Read)** | Yes | Yes | Yes | Yes |
| **Product Master (Create / Edit / Archive)** | No | No | No | Yes |
| **Product Stock Counts (`available_stock`)** | Read (Public) | Read | Read | Read |
| **Inventory Movements Ledger (Read)** | No | No | No | Yes |
| **Manual Stock Adjustments (`adjust_stock`)** | No | No | No | Yes |
| **Suppliers Directory (Read / Write)** | No | **No (Blocked)** | No | Yes |
| **Purchase Orders & Goods Receipt** | No | **No (Blocked)** | No | Yes |
| **Product Cost Price / Purchase Invoices** | No | **No (Blocked)** | No | Yes |
| **Sales Counter & Billing** | No | Yes | No | Yes |
| **Customer Credit Ledger (Udhaar) (Read)** | Own Balance | Read Balance | No | Full Read |
| **Modify Customer Credit Limits** | No | No | No | Yes |
| **Online Bookings (Create)** | Yes (Own / Guest) | Yes (Walk-in) | No | Yes |
| **Online Bookings (Transition / Fulfill)** | Cancel Own | Yes | Assigned Only | Yes |
| **Delivery Orders (View & Dispatch)** | Own Orders | View All | Assigned Only | Full Control |
| **Financial & Profit Reports** | No | **No (Blocked)** | No | Yes |
| **Shop Settings (Licenses, Tax, Bank Info)** | Read Public Info | Read Public Info | Read Public Info | Full Control |
| **User Role Assignment** | No | No | No | Yes |

---

## 3. Role Assignment Security

- **Self-Assignment Prohibition**: Users can **never** assign or modify roles for themselves.
- **Implementation**:
  - Roles are stored in a dedicated `user_roles` table (`user_id`, `role`, `created_at`, `assigned_by`) outside the editable `profiles` table.
  - The `user_roles` table is protected by PostgreSQL RLS with `INSERT`, `UPDATE`, and `DELETE` policies restricted strictly to authenticated users with the `admin` role.
  - User registration via Supabase Auth defaults automatically to the `customer` role via a database trigger.
  - Elevated roles (`employee`, `delivery`, `admin`) can only be provisioned through an explicit admin action or initial setup script during deployment.

---

## 4. PostgreSQL Row Level Security (RLS) Principles

1. **Default Deny Strategy**: Every single table created in the `public` schema has Row Level Security explicitly enabled (`ALTER TABLE ... ENABLE ROW LEVEL SECURITY;`). No query returns data unless an explicit policy permits it.
2. **Read-Only Inventory Exposure**:
   - The public/customer tier only reads calculated `available_stock` through an API view or sanitized column.
   - Direct `INSERT`, `UPDATE`, or `DELETE` on `inventory_movements`, `product_stock`, `sales`, and `bookings` is completely denied to all client roles. All mutations must occur via validated PostgreSQL stored functions.
3. **Data Isolation by Tenant / Customer**:
   - Customers can only query bookings, orders, and ledger entries matching their authenticated UID (`auth.uid() = user_id`).
   - Guest bookings are secured via non-guessable UUIDs and tokenized booking references.
4. **Role Verification Helper**:
   - A secure database function `auth.has_role(required_role text)` inspects the verified `user_roles` table or decoded JWT app metadata, cached within the session transaction.

---

## 5. SECURITY DEFINER Stored Procedure Rules

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

## 6. Personally Identifiable Information (PII) Protection

Pesticide retail involves sensitive customer personal data. The system categorizes and protects PII in accordance with Indian Digital Data Protection standards:

### Identified PII Data Elements
- Customer Full Name
- Primary Mobile Phone Number
- Village, Mandal, and District Details
- Street / Farm Delivery Address
- Purchase & Consumption History (crop types and pesticide usage)
- Credit Ledger Balances (Udhaar financial standing)

### Safeguards
- PII is restricted to store staff with an operational need.
- Delivery agents only receive customer name, phone, and delivery address for active dispatches; once delivered, historical contact details are masked.
- Customer credit balances are considered sensitive financial data and are hidden from public and delivery views.
- No PII is logged in application error trackers or client console outputs.

---

## 7. Progressive Web App (PWA) Caching Rules

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
