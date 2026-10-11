# Architecture & Product Decisions Log

This document records the foundational product and architectural decisions for **Vighnesh AgroCare**.

## Decisions Summary Table

| ID | Decision | Choice | Status | Date |
| :--- | :--- | :--- | :--- | :--- |
| **D01** | Costing method | Weighted Average Cost (WAC) recalculated upon every goods receipt | Confirmed | 2026-10-11 |
| **D02** | Batch and expiry tracking | Batch number and expiry recorded on goods receipt; inventory balance deducted at product level; revisit batch-level stock in Phase 4 | Provisional | 2026-10-11 |
| **D03** | Product variant modeling | Each sellable pack size is modeled as its own independent product row in `products` | Confirmed | 2026-10-11 |
| **D04** | Pricing & tax calculation | Prices stored tax-inclusive with per-product tax rate; actual GST rates and invoice rules TO BE VERIFIED WITH ACCOUNTANT | TO BE VERIFIED WITH ACCOUNTANT | 2026-10-11 |
| **D05** | Customer authentication | Email/password first; guest bookings rate-limited with CAPTCHA; phone OTP deferred to later phase | Confirmed | 2026-10-11 |
| **D06** | Booking reservation expiry | 24 hours hold period, automatically released by a scheduled pg_cron / Edge Function job | Confirmed | 2026-10-11 |
| **D07** | Order notifications | Pre-filled `wa.me` click-to-chat WhatsApp link on frontend + manual staff confirmation | Confirmed | 2026-10-11 |
| **D08** | Multi-store vs single-store | Single shop initially; centralized `shop_settings` table for all shop configuration | Confirmed | 2026-10-11 |
| **D09** | Maximum Retail Price enforcement | Selling price must not exceed MRP (enforced strictly via PostgreSQL database constraint/function) | Confirmed | 2026-10-11 |
| **D10** | Credit ledger (Udhaar) | Customer credit ledger with individual credit limits included in core database design | Confirmed | 2026-10-11 |
| **D11** | System user roles | Four discrete roles: `customer`, `admin`, `employee`, `delivery` | Confirmed | 2026-10-11 |
| **D12** | Application languages | English and Telugu (i18n enabled from Phase 2; Telugu-capable web font required) | Confirmed | 2026-10-11 |
| **D13** | Jurisdiction & operating state | State of Andhra Pradesh, India | Confirmed | 2026-10-11 |
| **D14** | Environment isolation | Separate isolated Supabase projects for development and production environments | Confirmed | 2026-10-11 |
| **D15** | Product restriction flags | Product-level flags: `allow_online_booking`, `allow_home_delivery` (defaults to false for every product), `requires_manual_review` | Confirmed | 2026-10-11 |
| **D16** | Stock returns handling | Return movement types defined in schema now (`customer_return`, `supplier_return`); management screens built later | Confirmed | 2026-10-11 |
| **D17** | Production infrastructure tier | Supabase Pro tier (providing daily automated backups and point-in-time recovery) required prior to production launch | Confirmed | 2026-10-11 |
| **D18** | Business and product scope | Scope = pesticides only; fertilizers, seeds and subsidized inputs excluded | Confirmed | 2026-10-11 |
| **D19** | Initial product categories | Initial categories (configurable in the database): Insecticides, Fungicides, Herbicides, Acaricides, Nematicides, Rodenticides, Bio-pesticides | Confirmed | 2026-10-11 |
| **D20** | Pesticide regulatory & product data fields | Product records must support: CIB&RC registration number, manufacturer, technical name (active ingredient), formulation, toxicity colour class, batch number, manufacturing date, expiry date. Exact legally required fields TO BE VERIFIED WITH DEPARTMENT OF AGRICULTURE, ANDHRA PRADESH | Provisional | 2026-10-11 |

---

## Detailed Decision Notes

### D01: Costing Method
- **Rationale**: Pesticide retail items experience seasonal purchase price fluctuations. Strict FIFO/LIFO batch allocation at the counter requires scanning every single bottle/packet's batch during rapid counter sales, which slows down counter operations during peak pest outbreaks. Weighted Average Costing (WAC) provides accurate financial accounting and gross profit calculations while keeping counter billing fast.
- **Implementation**: On each purchase receipt, WAC is recomputed in the `receive_purchase` PostgreSQL function and persisted on the product record.

### D02: Batch & Expiry Tracking
- **Rationale**: Regulatory inspection compliance under the Insecticides Act, 1968 in Andhra Pradesh requires knowing which batches entered the shop and their manufacturer expiry dates. However, requiring counter staff to pick a specific batch during rapid retail sales of 50 farmers standing in line causes severe counter bottlenecks.
- **Implementation**: Batch number and expiry date are recorded when receiving goods from suppliers (`purchase_items`). Physical stock balances and deductions occur at the product level (`product_stock`). Batch-level audit history remains available from purchase records.
- **Status**: Provisional. Revisit batch-level stock in Phase 4.

### D03: Each Pack Size as an Independent Product Row
- **Rationale**: In agro-chemicals, a 100ml bottle, 250ml bottle, 500ml bottle, and 1L bottle of the same chemical formulation have different MRPs, different purchase costs, different supplier barcodes/HSN codes, and separate inventory counts. Grouping them as complex relational variants introduces schema complexity and makes stock lookups error-prone.
- **Implementation**: Each sellable SKU/pack size is its own row in the `products` table, with optional `parent_group_id` for catalog display grouping.

### D04: Pricing, Taxes & GST Rates
- **Rationale**: Retail customers in India expect prices displayed to be the final payable amount (MRP or discounted retail price inclusive of GST).
- **Status**: TO BE VERIFIED WITH ACCOUNTANT.
- **Pending Items**: Verification of specific pesticide HSN codes, applicable GST slabs (typically 18% on technical and formulated chemical pesticides under Andhra Pradesh GST rules), tax invoice formats, and B2C vs B2B threshold requirements under Andhra Pradesh GST rules.

### D05: Customer Authentication Strategy
- **Rationale**: Indian SMS gateways require DLT (Distributed Ledger Technology) registration, sender ID whitelisting, and per-SMS fees which add friction during the initial project bootstrap.
- **Implementation**: Initial phase launches with standard Supabase Email/Password authentication. Guest customers can place bookings by entering their name, phone number, and village, protected by Cloudflare Turnstile / hCaptcha rate-limiting. Phone OTP authentication via SMS provider (e.g. MSG91, Twilio) will be added in a subsequent phase.

### D06: 24-Hour Booking Reservation Expiry
- **Rationale**: High-demand seasonal crop protection products (e.g., pink bollworm sprays during cotton season or brown planthopper sprays during paddy outbreaks) must not remain blocked indefinitely by abandoned online bookings.
- **Implementation**: Bookings lock stock in `reserved_stock` for 24 hours. A scheduled cron job queries `bookings` where `status = 'pending'` and `created_at < NOW() - INTERVAL '24 hours'`, systematically transitioning them to `cancelled` and releasing reserved stock back to `available_stock`.

### D07: WhatsApp Communication via `wa.me`
- **Rationale**: Avoids costly third-party WhatsApp Business API setup and recurring Meta messaging charges during early retail operations.
- **Implementation**: When a customer confirms a booking or when staff confirms availability, the frontend generates a `wa.me/91XXXXXXXXXX?text=...` link with pre-populated order details, allowing instant one-tap messaging directly in WhatsApp.

### D08: Single Shop Operation with Settings Table
- **Rationale**: The project serves a dedicated retail outlet in Andhra Pradesh. Multi-tenant architecture would add unnecessary complexity at this stage.
- **Implementation**: A single `shop_settings` table stores shop name, address, contact numbers, pesticide retail dealership license numbers, GSTIN, UPI VPA, and working hours.

### D09: Maximum Retail Price (MRP) Enforcement
- **Rationale**: Under the Legal Metrology Act and Indian consumer laws, selling any packaged commodity above the printed Maximum Retail Price (MRP) is illegal and incurs statutory penalties.
- **Implementation**: A PostgreSQL check constraint (`selling_price <= mrp`) and transactional validation inside `complete_sale` strictly reject any transaction where unit price exceeds MRP.

### D10: Customer Credit Ledger (Udhaar Khata)
- **Rationale**: Credit sales are an essential commercial reality of agricultural input retail in rural India. Farmers purchase crop protection inputs during pest attacks and settle accounts at harvest.
- **Implementation**: Customer accounts feature an append-only `customer_credit_ledger` with an admin-configured `credit_limit`. Counter sales can be debited against credit, and debt repayments are explicitly recorded.

### D11: Four Discrete System Roles
- **Rationale**: Separation of duties between owners, staff, and field delivery agents.
- **Roles**:
  - `admin`: Shop owner; full access to all operations, purchase costs, supplier records, credit limits, and profit reports.
  - `employee`: Counter sales assistant; access to catalog, counter sales, stock lookup, booking processing; blocked from seeing purchase costs, supplier master, and profit reports.
  - `delivery`: Field delivery agent; access only to assigned deliveries, delivery addresses, and delivery payment status.
  - `customer`: Public user; access to catalog, own shopping cart, own bookings, and own profile.

### D12: Bilingual Support (English & Telugu)
- **Rationale**: The customer base is located in Andhra Pradesh. Telugu is the primary language for local farmers, while English is used for trade names, chemical formulations, and administrative tasks.
- **Implementation**: Internationalization (i18n) framework initialized in Phase 2 with a clean key-value dictionary and high-legibility Telugu typography (e.g. Noto Sans Telugu).

### D13: Operating Jurisdiction (Andhra Pradesh, India)
- **Rationale**: Insecticides Act retail licensing, state quality-control inspections, and agricultural department reporting guidelines are state-specific under the Government of Andhra Pradesh.

### D14: Environment Separation
- **Rationale**: Strict isolation between staging/testing and live customer orders, financial ledger entries, and live inventory counts.
- **Implementation**: Independent Supabase cloud instances for development and production, governed by distinct environment configuration files.

### D15: Product-Level Operational Flags
- **Rationale**: Certain hazardous agrochemicals require in-person identity verification or cannot be legally transported via home delivery; other seasonal items should only be sold over the counter.
- **Implementation**:
  - `allow_online_booking` (boolean, default true)
  - `allow_home_delivery` (boolean, defaults to false for every product)
  - `requires_manual_review` (boolean, default false)

### D16: Return Stock Movement Types Defined Upfront
- **Rationale**: Schema migrations on live transactional tables are costly. Defining return movement types early guarantees ledger integrity when return interfaces are delivered.
- **Implementation**: `customer_return` and `supplier_return` are valid movement types in the schema from day one.

### D17: Supabase Pro Tier Requirement
- **Rationale**: Point-in-time recovery (PITR) and daily automated database backups are indispensable for production financial ledgers, preventing catastrophic data loss from human error or operational failure.

### D18: Business and Product Scope
- **Rationale**: Crop protection chemicals (pesticides) constitute a distinct, highly regulated vertical under the Insecticides Act, 1968. Excluded from scope: chemical fertilizers, bio-fertilizers, seeds, subsidized inputs (such as Urea, DAP, MOP), and government Point-of-Sale (PoS) biometric Direct Benefit Transfer (DBT) systems. Focus is exclusively on retail pesticides.
- **Status**: Confirmed.

### D19: Initial Product Categories
- **Rationale**: Crop protection chemicals are classified by target pest, weed, or pathogen. Initial categories are configurable in the database to accommodate shop inventory needs.
- **Categories**: Insecticides, Fungicides, Herbicides, Acaricides, Nematicides, Rodenticides, Bio-pesticides.
- **Implementation**: Stored in a configurable `categories` database table so the shop owner can manage categories without code changes.
- **Status**: Confirmed.

### D20: Pesticide Regulatory & Product Data Fields
- **Rationale**: Under the Insecticides Act, 1968 and Insecticides Rules, 1971, agrochemical products must be strictly identified by approved technical formulations, manufacturer credentials, toxicity hazard levels, and batch lifecycles.
- **Specification**: Product records must support:
  - CIB&RC registration number (Central Insecticides Board & Registration Committee)
  - Manufacturer name
  - Technical name (active ingredient, e.g. Chlorpyrifos 20% EC, Imidacloprid 17.8% SL)
  - Formulation type (e.g. EC, WP, SC, SL, GR, WG, SP)
  - Toxicity colour class (Red: Extremely Toxic, Yellow: Highly Toxic, Blue: Moderately Toxic, Green: Slightly Toxic)
  - Batch number
  - Manufacturing date
  - Expiry date
- **Status**: Provisional. Exact legally required fields TO BE VERIFIED WITH DEPARTMENT OF AGRICULTURE, ANDHRA PRADESH.
