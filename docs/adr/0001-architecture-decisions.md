# ADR 0001: Encapsulating Business Logic in PostgreSQL Functions

## Status
**Accepted** (2026-10-11)

## Context
**Vighnesh AgroCare** is a retail pesticide platform operating in Andhra Pradesh, India. The application manages critical operational domains:
1. High-concurrency inventory reservation for seasonal crop protection agrochemicals (insecticides, fungicides, herbicides).
2. Counter sales billing and Maximum Retail Price (MRP) statutory compliance.
3. Inventory accounting using the Weighted Average Cost (WAC) method and an immutable stock movements ledger.
4. Customer credit (udhaar) accounting with individual credit limits.

Modern web architectures often default to placing business logic either inside client-side JavaScript applications or in an intermediary Node.js/serverless backend tier, utilizing the database merely as a dumb relational store.

In our stack (React, Vite, Supabase, PostgreSQL), allowing the client browser application to coordinate multi-step transactions (e.g. read stock -> check stock -> compute subtotal -> write booking -> write stock movement) introduces severe vulnerabilities:
- **Concurrency & Race Conditions**: In high-demand scenarios (such as sudden pest outbreaks during cotton or paddy season), network latency between client reads and writes makes race conditions virtually inevitable, leading to overselling.
- **Client-Side Tampering**: Trusting the browser to enforce business invariants (such as `selling_price <= mrp`, credit limits, or stock deductions) creates significant security risks, as client-side code can be inspected, altered, or bypassed.
- **Network Overhead & Partial Failures**: Orchestrating multi-table operations over multiple HTTP roundtrips on unreliable 3G/4G cellular connections in rural Andhra Pradesh increases failure rates, leaving databases in half-written or inconsistent states.
- **Leaky Security Model**: Permitting direct write access from the client browser to core tables requires overly complex Row Level Security (RLS) policies that are difficult to audit and maintain.

## Decision
We will encapsulate all mission-critical, transactional, and state-altering business logic directly within **PostgreSQL stored functions (PL/pgSQL)**.

Specifically:
- Core operations (`create_booking`, `transition_booking`, `cancel_booking`, `receive_purchase`, `adjust_stock`, `complete_sale`, `record_payment`) are authored as PostgreSQL functions running with `SECURITY DEFINER` privileges.
- Every function explicitly sets `search_path = public, pg_temp;` and verifies the caller's identity and role.
- Concurrency control is achieved using database-level ACID transactions and explicit row-level locks (`SELECT ... FOR UPDATE`).
- Direct client `INSERT`, `UPDATE`, and `DELETE` access to transactional tables (`inventory_movements`, `product_stock`, `sales`, `bookings`, `customer_credit_ledger`) is completely revoked at the RLS and grant levels.
- The React client interacts with backend state modifications exclusively by invoking these stored functions via Supabase RPC (`supabase.rpc(...)`).
- Edge Functions (TypeScript/Deno) are reserved strictly for external integrations that require sensitive third-party API keys (such as CAPTCHA validation or future SMS gateways).

## Consequences

### Positive
- **Guaranteed Atomic Consistency**: All steps within a reservation, sale, or goods receipt either commit completely or roll back entirely within the database engine.
- **Zero Overselling / Concurrency Safety**: Row-level locks (`SELECT ... FOR UPDATE`) serialize concurrent requests at the database layer, eliminating race conditions.
- **Tamper-Proof Invariants**: Business rules (e.g. MRP ceiling, positive stock checks, credit limit caps) cannot be bypassed by modified client applications.
- **Reduced Network Latency**: Complex workflows execute in sub-millisecond local database time rather than requiring multiple HTTP roundtrips over cellular networks.
- **Simplified Client Code**: The React UI code focuses purely on presentation, user experience, and form validation, rather than complex transaction management.

### Negative / Trade-Offs
- **Dual Language Development**: The team must write and maintain both TypeScript (frontend) and PL/pgSQL (database procedures).
- **Migration & Versioning Discipline**: Changes to business logic require disciplined SQL migration management and database integration testing (via pgTAP/SQL test suites).
- **Tooling Learning Curve**: Developers must be comfortable with PostgreSQL procedural language debugging, triggers, and locking mechanics.
