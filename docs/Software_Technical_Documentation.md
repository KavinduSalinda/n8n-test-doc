# POS System — Software Technical Documentation

**Product:** Point of Sale (POS) Platform  
**Version:** 1.0  
**Status:** PRD-derived Technical Architecture  
**Audience:** Developers, QA Engineers, DevOps Engineers, Project Managers, Technical Leads  
**Technology Scope:** POS Terminal, Web Admin, REST API, Database, Payments, Inventory, Hardware, Cloud  
**Last Updated:** September 2026

---

## Table of Contents

1. [Product Overview (PRD)](#1-product-overview-prd)
2. [Business Objectives](#2-business-objectives)
3. [Functional Requirements](#3-functional-requirements)
4. [Non-Functional Requirements](#4-non-functional-requirements)
5. [System Architecture](#5-system-architecture)
6. [POS Terminal Application](#6-pos-terminal-application)
7. [Backend and REST API](#7-backend-and-rest-api)
8. [Database Design](#8-database-design)
9. [Authentication and Authorization](#9-authentication-and-authorization)
10. [Multi-Store Architecture](#10-multi-store-architecture)
11. [Inventory and Catalog](#11-inventory-and-catalog)
12. [Sales and Checkout](#12-sales-and-checkout)
13. [Payments and Tender](#13-payments-and-tender)
14. [Hardware Integrations](#14-hardware-integrations)
15. [Notifications and Receipts](#15-notifications-and-receipts)
16. [Reporting and Analytics](#16-reporting-and-analytics)
17. [DevOps and Deployment](#17-devops-and-deployment)
18. [Environment Configuration](#18-environment-configuration)
19. [Logging and Monitoring](#19-logging-and-monitoring)
20. [Security](#20-security)
21. [Testing and Quality Assurance](#21-testing-and-quality-assurance)
22. [Performance and Scalability](#22-performance-and-scalability)
23. [Backup and Disaster Recovery](#23-backup-and-disaster-recovery)
24. [API Documentation Example](#24-api-documentation-example)
25. [Troubleshooting Guide](#25-troubleshooting-guide)
26. [Glossary](#26-glossary)
27. [Conclusion](#27-conclusion)

---

# 1. Product Overview (PRD)

This document converts Product Requirements Document (PRD) intent for a Point of Sale platform into a technical architecture that engineering can implement, test, and operate.

The POS platform enables retailers to sell products and services at the counter, manage inventory across stores, accept multiple payment methods, print or email receipts, and give managers real-time visibility into sales and stock.

The product is designed for:

- Retail stores
- Restaurants and quick-service counters
- Multi-store chains
- Franchise or multi-tenant retail groups
- Counter staff, cashiers, store managers, and back-office administrators

## Core Capabilities (from PRD)

- Fast checkout and order capture
- Product catalog and barcode scanning
- Inventory tracking and stock adjustments
- Cash, card, wallet, and split tender payments
- Receipt printing, email, and reprint
- Customer profiles and loyalty
- Discounts, promotions, and tax calculation
- Cash drawer and end-of-day reconciliation
- Role-based access for cashier, supervisor, and admin
- Multi-store data isolation
- Offline-tolerant terminal operation with later sync
- Sales, inventory, and cash reports
- Hardware support for scanners, printers, and payment terminals

---

# 2. Business Objectives

The PRD defines the following business outcomes:

1. Reduce checkout time and queue length at the counter.
2. Eliminate manual price lookup and handwritten sale records.
3. Keep inventory accurate after every sale, return, and adjustment.
4. Accept modern payment methods without delaying the cashier.
5. Give store managers a reliable end-of-day cash and sales close.
6. Support multiple stores on one platform with isolated data.
7. Provide searchable sales history for refunds, disputes, and audits.
8. Reduce shrinkage through permissions, voids, and audit logs.
9. Enable consistent tax, discount, and receipt rules across locations.
10. Scale from a single register to a chain of stores without rewriting the core system.

---

# 3. Functional Requirements

## 3.1 User and Role Management

The platform must support:

- Create, update, activate, and deactivate staff accounts
- Assign store-scoped roles and permissions
- Cashier PIN or badge login on the terminal
- Password reset for back-office users
- Session timeout on idle terminals
- Track login, logout, and failed authentication events

## 3.2 Catalog and Product Management

The catalog module must support:

- Product create, update, and deactivate
- SKU, barcode, name, category, unit, and tax class
- Price lists and store-level price overrides
- Variants (size, color, or option sets)
- Bundles and modifiers where required
- Product search by name, SKU, or barcode
- Image and description for back-office and optional customer display

## 3.3 Inventory Management

The inventory module must support:

- On-hand quantity per store
- Stock reservation during an open ticket
- Decrement on completed sale
- Increment on return or voided sale
- Receiving, transfer, and adjustment documents
- Low-stock alerts
- Inventory search and filtering
- Audit history for quantity changes

## 3.4 Sales and Checkout

The checkout module must support:

- Create a new sale / ticket
- Add, update quantity, and remove line items
- Apply item or ticket discounts
- Apply promotions and coupons
- Calculate tax by jurisdiction and tax class
- Hold and recall tickets
- Void line items or entire tickets with supervisor approval
- Parked / suspended orders
- Returns and exchanges against an original receipt
- Sale notes and cashier reference

## 3.5 Payments

The payment module must support:

- Cash tender with change calculation
- Card present payments through a payment terminal
- Digital wallets where the acquirer supports them
- Split tender (for example cash + card)
- Partial payments on an open ticket
- Refunds against the original tender where supported
- Payment status, authorization, and capture references
- Failed, declined, and cancelled payment handling

## 3.6 Customers and Loyalty

The customer module must support:

- Walk-in (anonymous) sales
- Customer registration and profile updates
- Contact details and sale history
- Loyalty points earn and redeem
- Store credit / account balance where enabled
- Customer search at checkout

## 3.7 Cash Management

The cash module must support:

- Opening float per register
- Paid-in and paid-out transactions
- Cash drawer open events
- Mid-shift and end-of-day counts
- Expected vs counted variance
- Z-report / close-of-day lock

## 3.8 Reporting

Reporting must include:

- Real-time dashboard for today's sales
- Sales by product, category, cashier, and store
- Payment tender mix
- Returns, voids, and discounts
- Inventory valuation and movement
- Cash drawer variance
- Exportable daily and period reports

---

# 4. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Availability | Terminals must remain usable during brief network interruptions; cloud APIs target high availability with health checks |
| Performance | Barcode add-to-cart and payment status updates should feel immediate at the counter |
| Security | Secrets must not be stored in frontend code; card data must never touch merchant servers in raw form |
| Scalability | Architecture must support additional stores and registers without redesign |
| Maintainability | Catalog, checkout, payments, and inventory must be modular services |
| Usability | Terminal UI must be operable with touch, keyboard, and barcode scanner |
| Observability | Sales, payments, sync failures, and hardware errors must be logged |
| Reliability | A completed sale must not be lost; payment and inventory updates must be consistent |
| Compliance | PCI DSS scope reduction via hosted / terminal-based card capture |
| Compatibility | Support current Chromium-based POS browsers, tablets, and supported payment terminals |

---

# 5. System Architecture

The PRD requires a distributed POS architecture: a store-facing terminal, a back-office web application, a central API, a transactional database, and integrations for payments and hardware.

A typical sale flows as follows:

1. Cashier authenticates on the terminal.
2. Terminal loads store catalog, tax, and register context.
3. Cashier scans or searches products onto a ticket.
4. API validates prices, stock, discounts, and tax.
5. Cashier collects tender.
6. Payment service talks to the acquirer or records a cash tender.
7. API commits the sale, inventory movement, and payment in one business transaction.
8. Receipt is printed, emailed, or stored for reprint.
9. Dashboard and reports read the committed sale.

## 5.1 Logical Architecture

```text
+------------------+     +-------------------+     +------------------+
| POS Terminal     |     | Back-Office Web   |     | Customer Display |
| (Cashier UI)     |     | (Admin / Manager) |     | (Optional)       |
+--------+---------+     +---------+---------+     +--------+---------+
         |                         |                         |
         +------------+------------+------------+------------+
                      |
                      v
              +-------+--------+
              | API Gateway /  |
              | HTTPS Ingress  |
              +-------+--------+
                      |
                      v
              +-------+--------+
              | POS REST API   |
              | Auth · RBAC    |
              +-------+--------+
                      |
      +---------------+---------------+----------------+
      |               |               |                |
      v               v               v                v
+-----+-----+  +------+-----+  +------+-----+  +------+------+
| Checkout  |  | Inventory  |  | Payments   |  | Reporting   |
| Service   |  | Service    |  | Service    |  | Service     |
+-----+-----+  +------+-----+  +------+-----+  +------+------+
      |               |               |                |
      +---------------+-------+-------+----------------+
                              |
                              v
                    +---------+---------+
                    | PostgreSQL        |
                    | Redis (cache/jobs)|
                    +---------+---------+
                              |
              +---------------+---------------+
              |               |               |
              v               v               v
      +-------+------+ +------+------+ +------+-------+
      | Payment      | | Email / SMS | | Hardware     |
      | Acquirer     | | Provider    | | Agent        |
      +--------------+ +-------------+ +--------------+
```

### Client Layer

- POS terminal web or desktop shell
- Back-office React (or equivalent) admin
- Optional customer-facing display
- Optional mobile manager app

### API Layer

- REST API for terminals and admin
- Authentication and authorization
- Idempotent sale and payment endpoints
- Store and register context resolution

### Business Layer

- Checkout and tax engine
- Inventory reservation and movement
- Discount and promotion rules
- Cash management
- Permissions and supervisor overrides

### Persistence Layer

- PostgreSQL for transactional data
- Redis for session, cache, and job queues
- Object storage for receipts, logos, and exports

### Integration Layer

- Card payment terminals / acquirers
- Email and SMS for receipts
- Accounting or ERP export
- Hardware agent for printer, drawer, and scanner

### Infrastructure Layer

- Linux hosts or containers
- Nginx or equivalent reverse proxy
- Background workers
- Cloud or on-premise deployment

## 5.2 Physical / Store Architecture

Each store has one or more registers. Each register is a POS terminal bound to a store, a cash drawer, and optional hardware.

```text
Store
 ├── Register 1 (Terminal + Drawer + Printer + Scanner)
 ├── Register 2
 ├── Local hardware agent (optional)
 └── Network to central API

Central Platform
 ├── API and workers
 ├── Database
 ├── Admin web
 └── Payment / email integrations
```

## 5.3 Offline and Sync Architecture

The PRD requires the counter to keep selling if the WAN link drops for a short period.

Rules:

- Terminal caches catalog, tax, and staff PINs for the assigned store.
- Cash sales may be completed locally and queued for sync.
- Card payments require the payment terminal path; if the acquirer is unreachable, the cashier is blocked from that tender.
- Each offline sale has a client-generated idempotency key.
- Sync worker uploads queued sales in order and applies inventory on the server.
- Conflicts (unknown SKU, negative stock policy) are flagged for manager review, not silently dropped.

```text
Offline Sale
      |
      v
Local Ticket Store
      |
      v
Sync Queue
      |
      v
POST /api/v1/sales/  (Idempotency-Key)
      |
      v
Server commit + inventory + receipt
```

---

# 6. POS Terminal Application

The terminal is the cashier's primary workspace. It must stay simple, fast, and recoverable after interruption.

Responsibilities:

- Staff login and register open
- Product search and barcode intake
- Ticket editing
- Discount and tax display
- Tender collection
- Receipt trigger
- Hold / recall / void flows
- Offline queue indicator

## 6.1 Recommended Structure

```text
src/
├── components/
├── pages/
│   ├── login/
│   ├── register-open/
│   ├── checkout/
│   ├── payments/
│   ├── returns/
│   └── close-day/
├── services/
├── hardware/
├── store/          # local ticket + offline queue
├── hooks/
├── utils/
├── constants/
├── layouts/
├── routes/
└── App.jsx
```

## 6.2 Terminal UX Constraints (PRD)

- Primary actions must be reachable in one or two taps.
- Scanner input must never steal focus from an unrelated text field.
- Line-item price changes and voids require a visible reason and, when configured, a supervisor PIN.
- The UI must remain usable in landscape tablet and desktop register layouts.

## 6.3 API Services

Frontend API access should be centralized.

```javascript
const sale = await api.post("/sales/", payload, {
  headers: { "Idempotency-Key": ticketId }
});
```

Centralized services manage:

- Access tokens
- Store and register headers
- Idempotency keys
- Offline replay
- Hardware callbacks
- Error mapping for the cashier

---

# 7. Backend and REST API

The backend exposes POS and back-office functionality through versioned REST endpoints.

Each protected endpoint should:

1. Authenticate the staff or admin user.
2. Resolve store and register from server-side context.
3. Validate input.
4. Enforce role and object-level authorization.
5. Apply business rules (price, tax, stock, tender).
6. Persist changes atomically.
7. Return a predictable JSON response.

## 7.1 HTTP Methods

| Method | Purpose | Example |
|---|---|---|
| GET | Retrieve resources | `/api/v1/products/` |
| POST | Create resource | `/api/v1/sales/` |
| PUT | Replace resource | `/api/v1/products/123/` |
| PATCH | Partial update | `/api/v1/sales/123/` |
| DELETE | Deactivate resource | `/api/v1/products/123/` |

## 7.2 Core Endpoint Groups

| Group | Base Path | Purpose |
|---|---|---|
| Auth | `/api/v1/auth/` | Login, PIN unlock, token refresh |
| Catalog | `/api/v1/products/` | Products, barcodes, prices |
| Inventory | `/api/v1/inventory/` | Stock, adjustments, transfers |
| Sales | `/api/v1/sales/` | Tickets, complete, void, return |
| Payments | `/api/v1/payments/` | Tender, capture, refund |
| Customers | `/api/v1/customers/` | Profiles, loyalty |
| Cash | `/api/v1/cash-drawers/` | Open, pay-in, close |
| Reports | `/api/v1/reports/` | Sales and inventory reports |
| Hardware | `/api/v1/devices/` | Printer / terminal pairing |

## 7.3 API Design Principles

- Use store-scoped URLs or server-derived store context.
- Require idempotency keys on sale complete and payment capture.
- Return stable error codes the terminal can map to cashier messages.
- Paginate catalog, sales history, and reports.
- Never trust client-supplied price as the source of truth unless a supervisor override is recorded.
- Never accept a tenant or store ID from the client without matching the authenticated session.

## 7.4 Example Response

```json
{
  "id": "sale_01J8QK",
  "status": "completed",
  "totals": {
    "subtotal": "42.00",
    "tax": "3.36",
    "discount": "0.00",
    "grand_total": "45.36"
  },
  "message": "Sale completed successfully"
}
```

---

# 8. Database Design

A relational database is required because sales, inventory, and payments are strongly consistent financial transactions.

The schema should include:

- Normalization of catalog and parties
- Foreign keys from lines to sales and products
- Unique constraints on SKU, barcode, and receipt number
- Indexes for register close and product lookup
- Audit fields on every mutable business record
- Store (tenant location) on every operational record

## 8.1 Core Entities

| Entity | Typical Data |
|---|---|
| Organization | Merchant / tenant, status, configuration |
| Store | Location, timezone, tax jurisdiction |
| Register | Store, hardware profile, status |
| User | Identity, role, store assignments |
| Product | SKU, barcode, name, tax class, status |
| Price | Product, store, amount, effective dates |
| InventoryItem | Product, store, on-hand, reserved |
| Customer | Name, contact, loyalty balance, store/org |
| Sale | Store, register, cashier, status, totals |
| SaleLine | Sale, product, qty, price, tax, discount |
| Payment | Sale, tender type, amount, provider reference |
| Receipt | Sale, number, print/email status |
| CashDrawerSession | Register, opening float, close totals |
| StockMovement | Product, store, reason, quantity delta |
| AuditLog | Actor, action, entity, before/after, timestamp |

## 8.2 Common Audit Fields

```text
created_at
updated_at
created_by
updated_by
```

Soft deletion where history must be retained:

```text
is_deleted
deleted_at
deleted_by
```

## 8.3 Indexing

Indexes should be considered for:

- Store ID + sale timestamp
- Receipt number
- Product barcode / SKU
- Sale status
- Inventory (store_id, product_id)
- Payment provider reference
- Cashier ID + business date

Indexes must follow real query paths (checkout lookup, close-of-day, receipt search).

---

# 9. Authentication and Authorization

Authentication verifies who is at the register or in the back office.

Authorization decides whether that person may discount, void, refund, or close the drawer.

All checks are enforced on the API. The terminal only hides controls; it does not grant them.

## 9.1 Recommended Practices

- Hash back-office passwords with a modern algorithm.
- Use short-lived access tokens and protected refresh tokens.
- Support cashier PIN login bound to a register session.
- Expire idle terminal sessions.
- Rate-limit login and PIN attempts.
- Require supervisor elevation for voids, refunds, and price overrides.
- Record the acting user and approver on sensitive events.
- Revoke tokens when a staff member is deactivated.

## 9.2 Example Roles

```text
PLATFORM_ADMIN
MERCHANT_ADMIN
STORE_MANAGER
SUPERVISOR
CASHIER
INVENTORY_CLERK
VIEWER
```

| Action | Cashier | Supervisor | Store Manager |
|---|---|---|---|
| Create sale | Yes | Yes | Yes |
| Apply configured discount | Yes | Yes | Yes |
| Custom price override | No | Yes | Yes |
| Void completed sale | No | Yes | Yes |
| Refund | No | Yes | Yes |
| Close cash drawer | No | Yes | Yes |
| Edit catalog | No | No | Yes |

---

# 10. Multi-Store Architecture

The PRD requires one platform to serve many stores while isolating operational data.

```text
Organization A
 ├── Store 1
 │    ├── Registers
 │    ├── Inventory
 │    ├── Sales
 │    └── Cash sessions
 └── Store 2
      ├── Registers
      ├── Inventory
      ├── Sales
      └── Cash sessions

Organization B
 └── Store 1
      └── ...
```

## 10.1 Isolation Rules

- Every operational row belongs to an organization and, where applicable, a store.
- Store context comes from the authenticated register session, not from a client-supplied header alone.
- Catalog may be shared at organization level with store price and stock overrides.
- Store A must never read Store B sales, cash, or on-hand stock unless a manager has an explicit cross-store role.
- Transfers are first-class documents, not silent stock edits in two stores.

## 10.2 Example Query Pattern

```python
Sale.objects.filter(
    store=request.register_session.store
)
```

Tenant and store filters should be centralized so individual endpoints cannot omit them.

---

# 11. Inventory and Catalog

Inventory is store-scoped. Catalog is organization-scoped with optional store overrides.

## 11.1 Sale-Time Stock Rules

1. Adding a line reserves quantity when the store enforces stock control.
2. Completing a sale converts reservation into a committed movement.
3. Cancelling or voiding a ticket releases the reservation.
4. A return creates a movement back into on-hand, subject to return reason rules.
5. Negative stock is either blocked or allowed with a manager reason, per store policy.

## 11.2 Movement Types

| Type | Effect |
|---|---|
| SALE | Decrease on-hand |
| RETURN | Increase on-hand |
| RECEIVING | Increase on-hand |
| TRANSFER_OUT | Decrease source store |
| TRANSFER_IN | Increase destination store |
| ADJUSTMENT | Increase or decrease with reason |

---

# 12. Sales and Checkout

A sale is the system of record for a customer transaction.

Sale states:

```text
open -> tendering -> completed
                 \-> cancelled
completed -> returned (partial or full)
open -> voided (supervisor)
```

## 12.1 Totals Engine

The server recalculates:

- Line extended price
- Line and ticket discounts
- Tax (inclusive or exclusive per store)
- Tendered amount and change due
- Grand total

Client-displayed totals are informational. The committed sale uses server totals.

## 12.2 Idempotency

Completing a sale is not safe to retry without an idempotency key. Double-tap or offline replay must return the original completed sale rather than create a second charge or stock movement.

---

# 13. Payments and Tender

Cardholder data must not enter the POS API. The terminal or a certified P2PE device tokenizes the card. The API stores only tender type, amount, status, and acquirer references.

## 13.1 Tender Types

- CASH
- CARD_PRESENT
- CARD_NOT_PRESENT (back-office only, if enabled)
- WALLET
- STORE_CREDIT
- SPLIT (composition of the above)

## 13.2 Payment States

```text
pending -> authorized -> captured
                     \-> declined
                     \-> cancelled
captured -> refunded (partial or full)
```

## 13.3 Consistency Rule

A sale may move to `completed` only when:

- Sum of successful tenders >= server grand total, and
- Inventory movements for the sale are written in the same database transaction, and
- A receipt number is allocated.

If the acquirer captures money and the local commit fails, a reconciliation job matches provider references and either completes the sale or flags it for support.

---

# 14. Hardware Integrations

The PRD assumes a physical counter.

| Device | Responsibility |
|---|---|
| Barcode scanner | Keyboard-wedge or HID input of SKU/barcode |
| Receipt printer | Print sale, reprint, and cash reports |
| Cash drawer | Open on cash tender or authorized no-sale |
| Payment terminal | Collect card/wallet, return token/result |
| Customer display | Show line items and amount due |
| Scale (optional) | Send weight for catch-weight items |

## 14.1 Hardware Agent

A local agent (or OS-level bridge) keeps printer and drawer access off the public internet.

```text
POS Terminal
      |
      v
Local Hardware Agent
      |
      +--> Receipt printer
      +--> Cash drawer
      +--> Payment terminal
```

The API records device identity and last-known health. It does not stream raw printer bytes through the public cloud unless a supported cloud-print path is explicitly chosen.

---

# 15. Notifications and Receipts

Receipts are a legal and customer artifact, not only a print action.

Channels:

- Thermal / standard printer
- Email
- SMS with receipt link
- In-app reprint from sale history

| Event | Notification |
|---|---|
| Sale completed | Print and optional email/SMS receipt |
| Refund completed | Refund receipt |
| Low stock | Manager alert |
| Drawer variance above threshold | Manager alert |
| Password / PIN reset | Staff security message |

Receipt and email generation should run asynchronously after the sale is committed so a printer or provider outage does not roll back a paid sale.

---

# 16. Reporting and Analytics

Reports are read models. They must not lock checkout tables longer than necessary.

Recommended approach:

- Operational queries against indexed sale tables for the current business day
- Aggregated daily snapshots for historical dashboards
- Export jobs for accounting (CSV or accounting-system API)

Key metrics:

- Gross sales, net sales, returns
- Average ticket
- Units per transaction
- Tender mix
- Top products
- Stock-outs and adjustments
- Cash over/short

---

# 17. DevOps and Deployment

Production must separate:

- Application code
- Configuration
- Secrets
- Receipt files and exports
- Logs
- Database data

## 17.1 Typical Deployment

```text
Internet
   |
   v
HTTPS
   |
   v
Nginx
   |
   v
POS API + Admin
   |
   +----> PostgreSQL
   |
   +----> Redis
   |
   +----> Workers (receipts, sync replay, reconciliation)
   |
   +----> Payment and email providers
```

Store terminals connect outbound over HTTPS. No inbound port to the register is required for the standard cloud deployment.

## 17.2 Deployment Components

- Linux hosts or containers
- Application runtime (for example Node.js or Python)
- PostgreSQL
- Redis
- Background workers
- Nginx
- Object storage
- CI/CD pipeline

---

# 18. Environment Configuration

Development, staging, and production must use separate projects, databases, and payment credentials. Payment staging must use acquirer sandbox accounts.

```env
SECRET_KEY=your-secret-key
DEBUG=False
DATABASE_URL=postgresql://...
REDIS_URL=redis://...
ALLOWED_HOSTS=pos.example.com
CORS_ALLOWED_ORIGINS=https://terminal.example.com
PAYMENT_PROVIDER_KEY=...
PAYMENT_PROVIDER_WEBHOOK_SECRET=...
EMAIL_HOST=...
EMAIL_USERNAME=...
EMAIL_PASSWORD=...
RECEIPT_STORAGE_BUCKET=...
```

Rules:

- Never commit production secrets.
- Never put acquirer secrets in the terminal bundle.
- Disable debug in production.
- Use a secrets manager when available.
- Rotate staff and integration credentials on offboarding.

---

# 19. Logging and Monitoring

## 19.1 Application Logs

Track:

- Authentication failures and PIN lockouts
- Sale complete, void, and refund events
- Payment capture and webhook mismatches
- Inventory movement failures
- Offline sync retries
- Hardware agent errors
- Unhandled API exceptions

Do not log full card numbers, CVV, or unredacted PANs. Payment references only.

## 19.2 Infrastructure Monitoring

- API latency and error rate
- Database connections and slow queries
- Queue depth for receipts and sync
- Disk and backup job health
- Terminal heartbeat / last-seen register time

## 19.3 Health Check

```text
GET /api/health/
```

```json
{
  "status": "healthy",
  "database": "connected",
  "redis": "connected"
}
```

Public health endpoints must not expose versions, credentials, or internal hostnames.

---

# 20. Security

Security is continuous. A POS system handles money, identity, and in some regions personal customer data.

## 20.1 Application Security Checklist

- [ ] HTTPS only for terminal and admin
- [ ] Server-side authentication and RBAC
- [ ] Supervisor overrides recorded
- [ ] CSRF protection on cookie-based admin sessions
- [ ] CORS limited to known terminal and admin origins
- [ ] Rate limiting on auth and sale complete
- [ ] Parameterized queries / ORM protections
- [ ] Input validation on money, quantity, and barcode fields
- [ ] Secure HTTP headers
- [ ] No secrets in terminal source maps
- [ ] Secrets excluded from Git
- [ ] PCI scope reduced (no raw card data in API or logs)
- [ ] Production debug disabled
- [ ] Dependency updates
- [ ] Database backups and restore tests

## 20.2 Secret Types

- Application secret key
- Database credentials
- Token signing keys
- Payment provider keys and webhook secrets
- Email credentials
- Object storage keys

All live outside the repository.

---

# 21. Testing and Quality Assurance

## 21.1 Test Types

| Test Type | Purpose |
|---|---|
| Unit | Tax, discount, change-due, and stock math |
| Integration | Sale complete with inventory and payment records |
| API | Auth, validation, idempotency, store isolation |
| Hardware | Scanner, print, drawer kick (lab devices) |
| Payment sandbox | Approve, decline, timeout, and refund |
| Offline | Queue, replay, duplicate-key protection |
| Security | Cross-store access, privilege escalation |
| UAT | Cashier and manager journeys from the PRD |

## 21.2 Regression After Checkout Changes

- Login and register open
- Barcode add and quantity change
- Discount and tax totals
- Cash and card complete
- Split tender
- Hold / recall
- Void and refund
- Receipt reprint
- Inventory on-hand after sale and return
- Close-of-day variance
- Offline sale sync
- Cross-store isolation

---

# 22. Performance and Scalability

Checkout is latency-sensitive. Catalog browse and reports are not allowed to block sale complete.

- Index barcode and store-scoped sale queries.
- Cache store catalog snapshots; invalidate on price or product change.
- Paginate history and reports.
- Move receipt rendering, email, and heavy reports to workers.
- Keep sale complete to one short database transaction.

Horizontal scale path:

- Stateless API instances behind a load balancer
- PostgreSQL primary for writes
- Read replicas for historical reports if needed
- Redis for cache and queues

---

# 23. Backup and Disaster Recovery

- Automated PostgreSQL backups
- Encrypted off-host backup storage
- Retention aligned to finance and tax rules
- Regular restore rehearsals
- Documented register procedure if the API is down (offline cash mode)

### Recovery Point Objective (RPO)

How much committed sale data the business can afford to lose. For POS, RPO should be near-zero for completed cloud-synced sales.

### Recovery Time Objective (RTO)

How quickly terminals must accept sales again after an outage. Offline cash mode reduces effective RTO at the counter.

---

# 24. API Documentation Example

## Complete Sale

### Endpoint

```http
POST /api/v1/sales/
```

### Headers

```http
Authorization: Bearer <access_token>
Idempotency-Key: 9f3c2b1a-7d44-4c11-9a0e-2b6d1f0e8c31
```

### Request

```json
{
  "register_id": "reg_14",
  "customer_id": null,
  "lines": [
    {
      "product_id": "prd_88",
      "quantity": 2,
      "barcode": "0123456789012"
    }
  ],
  "discounts": [],
  "payments": [
    {
      "tender_type": "CASH",
      "amount": "45.36"
    }
  ],
  "notes": ""
}
```

### Parameters

| Property | Type | Required | Description |
|---|---|---|---|
| register_id | String | Yes | Register that owns the ticket; must match the session |
| customer_id | String | No | Loyalty customer; omit for walk-in |
| lines | Array | Yes | Products and quantities |
| lines[].product_id | String | Yes | Catalog identifier |
| lines[].quantity | Number | Yes | Units sold |
| lines[].barcode | String | No | Scanned code for audit |
| discounts | Array | No | Ticket-level discounts |
| payments | Array | Yes | Tenders to apply |
| notes | String | No | Cashier note |

### Success Response

```json
{
  "id": "sale_01J8QK",
  "receipt_number": "S1-20260903-0142",
  "status": "completed",
  "totals": {
    "subtotal": "42.00",
    "tax": "3.36",
    "discount": "0.00",
    "grand_total": "45.36"
  },
  "change_due": "0.00",
  "message": "Sale completed successfully"
}
```

### HTTP Status

```text
201 Created
```

Repeated requests with the same idempotency key return the original sale (typically `200 OK`).

### Possible Errors

```text
400 Bad Request
401 Unauthorized
403 Forbidden
409 Conflict          # stock, register state, or duplicate in-flight sale
402 Payment Required  # tender declined or insufficient
422 Unprocessable     # totals or tax recalculation failed
500 Internal Server Error
```

---

# 25. Troubleshooting Guide

| Problem | Possible Cause | Action |
|---|---|---|
| Barcode does not add an item | Unknown SKU or scanner focus | Verify catalog barcode and terminal focus |
| Sale complete returns 409 | Stock policy or duplicate submit | Check on-hand and idempotency key |
| Card payment pending | Terminal or acquirer timeout | Check device link and payment logs; do not resubmit without key |
| Receipt did not print | Printer / agent issue | Confirm hardware agent; reprint from history |
| Drawer did not open | Kick cable or non-cash tender | Verify drawer mapping; use authorized no-sale if needed |
| Offline sales missing | Sync queue or auth expiry | Inspect terminal queue and retry after login |
| Wrong tax | Store jurisdiction or tax class | Verify store tax config and product class |
| Cross-store product missing | Catalog not published to store | Check store assignment and price list |
| Close-of-day variance | Unrecorded pay-in/out or void | Review drawer events and tenders |
| API 500 | Unhandled exception | Check API logs and latest deploy |

---

# 26. Glossary

| Term | Meaning |
|---|---|
| POS | Point of Sale — hardware and software used to complete a retail sale |
| PRD | Product Requirements Document |
| Register | A terminal plus cash drawer identity inside a store |
| Ticket / Sale | A customer transaction in progress or completed |
| Tender | A payment method applied to a sale |
| SKU | Stock keeping unit |
| Idempotency key | Client token that prevents duplicate sale or payment commits |
| Z-report | End-of-day register close report |
| P2PE | Point-to-point encryption of card data |
| PCI DSS | Payment Card Industry Data Security Standard |
| RBAC | Role-Based Access Control |
| RPO | Recovery Point Objective |
| RTO | Recovery Time Objective |
| REST | Representational State Transfer |
| JWT | JSON Web Token |

---

# 27. Conclusion

This document turns the POS PRD into an implementable system architecture.

The platform is a store-scoped, API-centered POS: terminals run checkout, the server owns price, tax, stock, and money, and payments stay outside the merchant card-data scope. Offline cash capability, idempotent sale complete, supervisor controls, and close-of-day cash management are required for production retail use.

Engineering should implement against the entities, endpoint groups, and consistency rules in this document, then validate cashier, supervisor, and manager journeys from the original PRD before go-live.
