# Vighnesh AgroCare

**Repository**: `VA-Project`  
**Current Phase**: **Phase 1: Requirements, Architecture & Structural Foundation**

---

## 1. Project Purpose & Scope

**Vighnesh AgroCare** is a production-grade Progressive Web Application (PWA) designed specifically for a retail pesticide shop in **Andhra Pradesh, India**. The store specializes in retailing crop protection agrochemicals (insecticides, fungicides, herbicides, acaricides, nematicides, rodenticides, and bio-pesticides) to local farmers (see [Decision D18](docs/00-decisions-log.md) for business scope).

The application delivers two integrated interfaces:
1. **Farmer & Customer PWA (English & Telugu)**: A fast, lightweight, mobile-optimized experience allowing farmers to browse crop protection products, check real-time stock availability and Maximum Retail Prices (MRP), place 24-hour product reservations, and initiate local deliveries or shop pickups with one-tap WhatsApp integration.
2. **Owner & Staff Operations Portal (`/admin`)**: A comprehensive management dashboard providing fast counter sales and billing, goods receipt with batch and expiry logging, automated Weighted Average Cost (WAC) recalculation, customer credit (udhaar) ledger tracking, supplier management, and business analytics.

---

## 2. Technology Stack

The technology stack is finalized as follows:

| Layer | Technology Choice | Rationale |
| :--- | :--- | :--- |
| **Frontend Framework** | **React 18+ & TypeScript** | Component-driven architecture with strong typing |
| **Build & Tooling** | **Vite** | Sub-second HMR and optimized production bundles |
| **Styling & Design System** | **Tailwind CSS** | Utility-first CSS tailored with modern agricultural tokens |
| **Backend & Database** | **Supabase (PostgreSQL 15+)** | Managed PostgreSQL, Auth, Storage, and Realtime |
| **Business Logic Tier** | **PostgreSQL (PL/pgSQL) Stored Functions** | ACID transactions, row locking (`SELECT ... FOR UPDATE`), idempotency |
| **Edge Compute** | **Supabase Edge Functions (TypeScript/Deno)** | Used exclusively where external secrets or webhooks are needed |
| **Data Access & Security** | **PostgREST + Row Level Security (RLS)** | 100% default-deny security policies and role enforcement |
| **Unit & Integration Testing** | **Vitest & React Testing Library** | High-speed unit and component test runner |
| **End-to-End Testing** | **Playwright** | Concurrency, booking flow, and browser validation |
| **Hosting & Edge Delivery** | **Vercel** | Global CDN caching and static asset distribution |

---

## 3. Current Status: Phase 1 (Foundation & Structure)

> [!NOTE]
> **Phase 1 is strictly dedicated to requirements specification, system architecture, security modeling, and repository structuring.**
> No application code, Vite scaffolding, package dependencies, or database migrations are created in this phase.

### Foundation Checklist
- [x] Repository cloned and active on branch `phase-1-foundation`
- [x] Production `.gitignore` configured for Node, Vite, Supabase, and testing artifacts
- [x] Sanitized `.env.example` created with security warnings regarding service-role keys
- [x] Directory skeleton established with `.gitkeep` markers
- [x] Core documentation specifications completed in `docs/` and `docs/adr/`
- [x] Zero application source code, zero external dependencies, zero secrets committed

---

## 4. Documentation Index

Detailed architectural and operational documentation is available in the [`docs/`](docs/) directory:

- [**00-decisions-log.md**](docs/00-decisions-log.md): Record of foundational architectural and product decisions (D01–D20), covering scope (pesticides only), costing methods, reservation lifecycles, and role models.
- [**01-requirements.md**](docs/01-requirements.md): Comprehensive functional and non-functional requirements organized across 14 domain modules.
- [**02-architecture.md**](docs/02-architecture.md): System architecture overview, text architecture diagram, PL/pgSQL function specifications, and planned Phase 2 source tree.
- [**03-security-model.md**](docs/03-security-model.md): Role-Based Access Control (RBAC) matrix, Row Level Security (RLS) rules, PII classification, and PWA caching policies.
- [**04-inventory-rules.md**](docs/04-inventory-rules.md): Inventory accounting invariants (`physical`, `reserved`, `available`), append-only ledger movement types, 24-hour reservation lifecycle, and Weighted Average Costing (WAC) worked example.
- [**05-roadmap.md**](docs/05-roadmap.md): The 20 implementation phases with specific goals and acceptance criteria.
- [**06-test-strategy.md**](docs/06-test-strategy.md): Testing pyramid (unit, integration, e2e) and test plans for 9 critical edge cases (concurrency, race conditions, duplicate receipts, unauthorized access).
- [**07-legal-checklist.md**](docs/07-legal-checklist.md): Mandatory statutory and regulatory verification items under the Insecticides Act, 1968, Insecticides Rules, 1971, and GST rules for Andhra Pradesh pesticide retail.
- [**adr/0001-architecture-decisions.md**](docs/adr/0001-architecture-decisions.md): Architecture Decision Record detailing why core business logic is encapsulated in PostgreSQL stored functions rather than client-side code.

---

## 5. Security Warning

> [!CAUTION]
> **NEVER** place the Supabase `service_role` key, database connection passwords, or any sensitive private keys in client-side environment files (`.env`, `.env.local`) or commit them to source control. Service-role keys bypass PostgreSQL Row Level Security completely and are restricted strictly to secure, server-side environments.
