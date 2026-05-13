# SmartCart — Product Requirements Document

---

## Document Control

| Field        | Value                                      |
|--------------|--------------------------------------------|
| Document ID  | PRD-001                                    |
| Version      | 1.0.0                                      |
| Status       | Approved                                   |
| Author       | SmartCart Engineering                      |
| Created      | 2026-05-13                                 |
| Last Updated | 2026-05-13                                 |

### Change Log

| Version | Date       | Author              | Summary                  |
|---------|------------|---------------------|--------------------------|
| 1.0.0   | 2026-05-13 | SmartCart Engineering | Initial approved version |

### Traces To

This document is the root anchor for all subsequent design artifacts.
All ADRs, NFR commitments, HLD diagrams, OpenAPI specs, and AsyncAPI
schemas must trace back to requirements defined here.

---

## 1. Purpose

This document defines the complete product requirements for SmartCart —
a production-grade, AI-enhanced, multi-vendor B2C e-commerce marketplace.

It exists to answer one question before any code is written:
**What exactly is being built, for whom, and what does done look like?**

Every architectural decision, service boundary, and data model in this
project must be justifiable by a requirement in this document. If a
feature has no requirement here, it is not in scope.

---

## 2. Product Overview

SmartCart is a general-purpose B2C marketplace where independent vendors
list products and customers discover, buy, track, and review them.

It models the operational domain of a platform like Amazon Marketplace:
horizontal product categories, multiple independent sellers, a single
customer-facing storefront, and platform-level operations managed by
administrators.

### 2.1 Design Principles

**General marketplace, not niche.** SmartCart is horizontal — any
category, any vendor type. No domain specialisation is assumed.

**Auth-first, no guests.** Every actor must be authenticated before
transacting. All requests carry a valid JWT.

**Single currency.** All monetary values are denominated in Indian
Rupees (INR). Multi-currency is explicitly out of scope.

**Vendor-shipped fulfillment.** SmartCart is not a logistics provider.
Vendors ship orders independently and enter tracking information into
the platform. SmartCart tracks status, not physical movement.

**Admin-controlled vendor access.** Vendors do not self-activate.
Every vendor account requires explicit Admin approval before listing
products or receiving orders.

**AI features are first-class.** Personalised recommendations, semantic
search, image search, and RAG-powered chatbot are core product
features — not experimental additions.

---

## 3. Actors

SmartCart has exactly three actor types. Every user story belongs to
exactly one actor.

### 3.1 Customer

A registered end-user who browses the catalog, adds products to cart,
places orders, tracks fulfillment, and submits reviews.

**Goals:**
- Discover products relevant to their needs quickly
- Complete purchases with confidence that inventory and payment are handled correctly
- Track order status in real time without polling or refreshing
- Share feedback on products and vendors

**Frustrations with existing platforms:**
- Discovery relies entirely on keywords; semantic intent is ignored
- No conversational interface for product questions
- Order tracking requires leaving the platform to visit courier websites

### 3.2 Vendor

A registered business or individual seller, approved by an Admin, who
manages their product catalog, fulfills orders assigned to them, and
monitors their store performance.

**Goals:**
- List products with full variant support (size, color, material, etc.)
- Receive and fulfill the portion of customer orders that belong to them
- Track their own performance metrics (sales, ratings, revenue)

**Constraints:**
- A vendor can only see and operate on their own data — never another vendor's
- A vendor cannot activate their own account — Admin approval is required
- A vendor cannot modify orders; they can only update fulfillment status

### 3.3 Admin

A platform operator who manages vendor approvals, monitors platform
health, and intervenes in operational exceptions.

**Goals:**
- Review and approve or reject vendor applications
- Monitor platform-level metrics (GMV, active users, order volume)
- Investigate operational issues (failed sagas, fraud flags, DLQ alerts)
- Manage the product category taxonomy

---

## 4. Functional Requirements

Requirements are grouped by feature area. Each requirement has a unique
ID in the format `FR-<AREA>-<NUMBER>`.

---

### 4.1 Authentication and Account Management

#### FR-AUTH-001 — Customer Registration
The system shall allow any unauthenticated user to register a customer
account using email address and password.

**Acceptance criteria:**
- Email must be unique across all account types
- Password must meet minimum strength requirements (min 8 characters,
  at least one uppercase, one number, one special character)
- Password is hashed using bcrypt cost ≥12 or argon2id before storage
- Registration sends a verification email to the provided address
- Account cannot be used to transact until email is verified
- Duplicate email registration returns a 409 Conflict with Problem Details body

#### FR-AUTH-002 — Vendor Registration
The system shall allow any unauthenticated user to apply for a vendor
account. Vendor accounts require Admin approval before activation.

**Acceptance criteria:**
- Applicant provides: business name, email, password, GST number
  (optional), business description
- Application is stored with status `PENDING_APPROVAL`
- Applicant receives a confirmation email acknowledging receipt
- Account cannot list products or receive orders until status
  transitions to `APPROVED`
- Duplicate email registration returns a 409 Conflict

#### FR-AUTH-003 — Login
The system shall authenticate registered users with email and password
and issue a short-lived access token and a longer-lived refresh token.

**Acceptance criteria:**
- Successful login returns: access token (JWT, RS256, TTL ≤15 min),
  refresh token (opaque or JWT, TTL ≤7 days)
- Failed login with wrong password returns 401 Unauthorized
- Rate limiting: maximum 10 failed login attempts per 15 minutes
  per IP; after threshold, subsequent attempts return 429
- Login for a PENDING or REJECTED vendor account returns 403 Forbidden
  with a descriptive error message

#### FR-AUTH-004 — Token Refresh
The system shall allow clients to obtain a new access token using a
valid refresh token without requiring re-authentication.

**Acceptance criteria:**
- Valid refresh token returns a new access token and a new refresh token
  (rotation on use)
- Used refresh token is invalidated immediately on rotation
- Expired or invalid refresh token returns 401 Unauthorized
- Refresh token rotation is atomic — no window where both old and new
  are simultaneously valid

#### FR-AUTH-005 — Logout
The system shall allow authenticated users to invalidate their current
session.

**Acceptance criteria:**
- Logout adds the access token's JTI to a Redis blocklist with TTL
  equal to remaining token lifetime
- Logout invalidates the refresh token server-side
- Subsequent requests with the invalidated access token return 401
- Logout is idempotent — logging out twice does not return an error

#### FR-AUTH-006 — Role-Based Access Control
The system shall enforce RBAC at the service layer, not only at the
gateway or UI.

**Acceptance criteria:**
- Three roles: CUSTOMER, VENDOR, ADMIN
- Each role has a disjoint set of permitted operations documented
  per service
- A CUSTOMER cannot access vendor portal or admin dashboard endpoints
- A VENDOR cannot access admin dashboard endpoints
- A VENDOR can only access their own data — cross-vendor access returns
  403 Forbidden
- RBAC violations are logged as security events with userId and
  attempted resource

---

### 4.2 Vendor Onboarding and Management

#### FR-VENDOR-001 — Vendor Application Review
The system shall provide Admins with the ability to review, approve, or
reject pending vendor applications.

**Acceptance criteria:**
- Admin can view a paginated list of applications filtered by status:
  PENDING_APPROVAL, APPROVED, REJECTED
- Admin can view full application details for any applicant
- Admin can approve an application with an optional note
- Admin can reject an application with a mandatory rejection reason
- On approval: vendor account status transitions to APPROVED, vendor
  receives an approval email, vendor can immediately log in and list
  products
- On rejection: vendor account status transitions to REJECTED, vendor
  receives a rejection email containing the rejection reason

#### FR-VENDOR-002 — Vendor Profile Management
The system shall allow approved vendors to manage their public store
profile.

**Acceptance criteria:**
- Vendor can update: business name, store description, logo image,
  contact email (display only — login email is immutable)
- Store profile is publicly visible on the customer storefront
- Store profile displays vendor's aggregate rating (computed from
  vendor reviews)
- Logo image is stored in MinIO and served via a CDN-compatible URL

#### FR-VENDOR-003 — Vendor Store Page
The system shall provide a public-facing vendor store page on the
customer storefront.

**Acceptance criteria:**
- Page displays: vendor name, description, logo, aggregate rating,
  total review count, and all active product listings by this vendor
- Page is accessible without authentication (browse only — adding to
  cart requires login)

---

### 4.3 Product Catalog Management

#### FR-CATALOG-001 — Product Creation
The system shall allow approved vendors to create product listings.

**Acceptance criteria:**
- A product has: name, description, category (from platform taxonomy),
  brand (optional), images (1–10), tags (for search), and one or more
  variants
- A product must have at least one variant before it can be published
- Product is created in DRAFT status — not visible to customers
- Vendor can save a product as DRAFT and return to complete it later

#### FR-CATALOG-002 — Product Variants
The system shall support product variants with independent pricing and
inventory per variant.

**Acceptance criteria:**
- A variant is defined by one or more attribute-value pairs:
  e.g. `{size: "M", color: "Red"}`
- Each variant has its own: SKU (auto-generated or vendor-provided),
  price in INR, and stock quantity
- A product can have at most 3 variant dimensions
  (e.g. size + color + material)
- Each variant dimension can have at most 20 values
- SKU must be unique per vendor (not globally)
- Price must be > ₹0 and stored as NUMERIC(19,4)

#### FR-CATALOG-003 — Product Publishing
The system shall allow vendors to publish and unpublish products.

**Acceptance criteria:**
- Vendor can transition a complete DRAFT product to ACTIVE status
- ACTIVE products are visible to customers and searchable
- Vendor can transition an ACTIVE product to INACTIVE status
- INACTIVE products are not visible to customers but order history
  referencing them is preserved
- A product cannot be published if it has zero stock across all variants
- Published products are indexed in Elasticsearch within 30 seconds

#### FR-CATALOG-004 — Product Updates
The system shall allow vendors to update product details after publishing.

**Acceptance criteria:**
- Vendor can update: name, description, images, tags, and variant prices
- Stock quantity updates flow through the Inventory service, not
  directly on the catalog
- Category changes require re-indexing in Elasticsearch
- Updates to published products are reflected in search within 30 seconds

#### FR-CATALOG-005 — Category Taxonomy
The system shall maintain a hierarchical product category tree managed
by Admins.

**Acceptance criteria:**
- Categories are hierarchical with maximum 3 levels: e.g.
  Electronics → Mobile Phones → Smartphones
- Admin can create, rename, and deactivate categories
- Deactivating a category does not delete existing products in that
  category — products are marked as requiring re-categorisation
- At least 10 seed categories are pre-populated at system initialisation

#### FR-CATALOG-006 — Product Image Management
The system shall allow vendors to upload and manage product images.

**Acceptance criteria:**
- Accepted formats: JPEG, PNG, WebP
- Maximum image size: 5 MB per image
- Maximum 10 images per product
- Images are stored in MinIO
- At least one image is required before a product can be published
- First image is designated as the primary (thumbnail) image

---

### 4.4 Product Discovery

#### FR-DISC-001 — Keyword Search
The system shall allow customers to search for products by keyword.

**Acceptance criteria:**
- Search query matches against: product name, description, brand, tags,
  category name
- Results are ranked by relevance score from Elasticsearch
- Results can be filtered by: category, price range (min/max INR),
  vendor rating (minimum stars), in-stock only
- Results can be sorted by: relevance (default), price ascending,
  price descending, newest, average rating
- Pagination is cursor-based; default page size is 20
- Only ACTIVE products from APPROVED vendors appear in results
- Search responds within p99 < 500 ms at sustained load

#### FR-DISC-002 — Semantic Search
The system shall support semantic search — finding products that match
the *intent* of a query even when keywords do not literally match.

**Acceptance criteria:**
- Query "comfortable running footwear" returns sports shoes even if the
  word "shoe" is absent from the query
- Semantic results are blended with keyword results — not a separate
  search mode; customers use one search bar
- Vector embeddings for all ACTIVE products are maintained in Qdrant
- Embeddings are updated within 60 seconds of a product update

#### FR-DISC-003 — Image Search
The system shall allow customers to search for products by uploading
an image.

**Acceptance criteria:**
- Customer uploads an image (JPEG, PNG, WebP, max 5 MB)
- System returns products visually similar to the uploaded image
- Image search results use the same filter and sort options as keyword
  search
- Image search responds within p99 < 3 s
- A clear "search by image" entry point exists on the storefront
  search bar

#### FR-DISC-004 — Personalised Recommendations
The system shall display personalised product recommendations to
authenticated customers.

**Acceptance criteria:**
- Homepage displays a "Recommended for you" section with up to 12
  products
- Recommendations are based on: browsing history, purchase history,
  and collaborative filtering signals from similar users
- New customers with no history receive popularity-based
  recommendations (cold start fallback)
- Recommendations exclude products the customer has already purchased
- Recommendation section loads within p99 < 500 ms (inference cache
  backed by Redis)

#### FR-DISC-005 — RAG Chatbot
The system shall provide a conversational AI assistant that can answer
product questions using the live product catalog.

**Acceptance criteria:**
- Chatbot is accessible from all pages via a persistent widget
- Customer can ask natural language questions:
  e.g. "What laptops do you have under ₹50,000?"
  e.g. "Is this product available in blue?"
  e.g. "What is the return policy for electronics?"
- Chatbot retrieves relevant product and policy documents via RAG
  before generating a response
- Chatbot can link to specific product pages in its response
- Chatbot can retrieve a customer's own order status when asked
  (JWT forwarded to Order service)
- Responses stream token-by-token (first token p99 < 2.5 s)
- Conversation history is maintained within a session (Redis-backed)
- Chatbot clearly identifies itself as an AI assistant
- Chatbot does not fabricate product details — it answers only from
  retrieved context or declines to answer

#### FR-DISC-006 — Product Detail Page
The system shall display a full product detail page for each ACTIVE
product.

**Acceptance criteria:**
- Page displays: all images (gallery), name, brand, category breadcrumb,
  description, vendor name (linked to vendor store page), all variants
  with current price and stock status per variant
- Out-of-stock variants are displayed but cannot be added to cart
- Average rating and total review count displayed with a link to reviews
- "Customers also viewed" section using Recommendation service
- Page loads within p99 < 500 ms

---

### 4.5 Cart Management

#### FR-CART-001 — Add to Cart
The system shall allow authenticated customers to add product variants
to their cart.

**Acceptance criteria:**
- Customer selects a variant before adding to cart
- Adding an out-of-stock variant returns a 409 Conflict
- Adding a variant already in the cart increments its quantity rather
  than creating a duplicate line
- Cart is persisted in Redis under the customer's userId
- Add-to-cart responds within p99 < 300 ms (NFR-PERF-005)
- Cart displays the current price at time of add — price changes after
  adding do not silently update the cart

#### FR-CART-002 — Cart Management
The system shall allow customers to view and modify their cart.

**Acceptance criteria:**
- Customer can view all cart items: product name, variant attributes,
  unit price, quantity, line total, vendor name
- Customer can change quantity of any cart item
- Customer can remove any cart item
- Customer can clear the entire cart
- Cart displays an order summary: subtotal, estimated total (no tax
  in scope), item count
- Cart is not session-scoped — it persists across logins and devices
  for the same userId

#### FR-CART-003 — Cart Validity
The system shall warn customers when cart items are no longer available.

**Acceptance criteria:**
- When a customer views their cart, the system checks current stock
  for each line item via gRPC call to Inventory service
- Cart items whose variant is now out of stock are flagged with a
  warning — they are not silently removed
- Cart items whose product is now INACTIVE are flagged with a warning
- Customer cannot proceed to checkout if any flagged items remain in
  cart (they must remove them first)

---

### 4.6 Checkout and Payment

#### FR-CHECKOUT-001 — Checkout Initiation
The system shall allow a customer to initiate checkout from their cart.

**Acceptance criteria:**
- Customer provides a shipping address before proceeding
- System validates cart has at least one item with sufficient stock
- System creates an order with status PENDING and fans out to N vendor
  sub-orders (one per vendor represented in the cart)
- Idempotency key is generated for the checkout session and returned
  to the client
- Checkout initiation (synchronous portion) responds within p99 < 2 s
  (NFR-PERF-007)

#### FR-CHECKOUT-002 — Inventory Reservation
The system shall reserve stock for all cart items atomically as part
of checkout.

**Acceptance criteria:**
- Inventory reservations are attempted for all line items before
  payment is initiated
- If any reservation fails (insufficient stock), the entire checkout
  is aborted and all successful reservations are released
- Reservations automatically expire after 15 minutes if payment is
  not completed (NFR-REL-004)
- Customer is informed which item caused the failure

#### FR-CHECKOUT-003 — Payment
The system shall process payment through a mock payment gateway.

**Acceptance criteria:**
- Payment is initiated via a REST call from Order service to Payment
  service with the order total and idempotency key
- Payment service calls the mock gateway and receives an async webhook
  callback with the result
- Webhook payload is verified using HMAC-SHA256 signature before
  processing (NFR-REL-009)
- Mock gateway simulates the following outcomes with configurable
  probability:
  1. SUCCESS — payment captured
  2. INSUFFICIENT_FUNDS — customer account balance too low
  3. CARD_DECLINED — gateway declined the card
  4. GATEWAY_TIMEOUT — mock gateway fails to respond within timeout
  5. FRAUD_HOLD — transaction flagged for manual review
- Each outcome triggers a distinct saga compensation path (see
  Section 4.6 FR-CHECKOUT-004)
- Payment service stores idempotency key — duplicate payment requests
  return the cached result without re-charging (NFR-REL-001)

#### FR-CHECKOUT-004 — Order Saga Outcomes
The system shall handle all payment outcomes with explicit compensation.

**Acceptance criteria:**

| Gateway Outcome    | Order Status     | Inventory       | Customer Notification     |
|--------------------|------------------|-----------------|---------------------------|
| SUCCESS            | CONFIRMED        | Reservation committed | "Order confirmed" email + WebSocket |
| INSUFFICIENT_FUNDS | PAYMENT_FAILED   | Reservation released | "Payment failed: insufficient funds" email |
| CARD_DECLINED      | PAYMENT_FAILED   | Reservation released | "Payment failed: card declined" email |
| GATEWAY_TIMEOUT    | PAYMENT_FAILED   | Reservation released | "Payment failed: please retry" email + idempotency key returned |
| FRAUD_HOLD         | UNDER_REVIEW     | Reservation held (not released, not committed) | "Order under review" email |

- All compensation steps are idempotent (NFR-REL-002)
- Saga completion (from checkout submit to WebSocket notification)
  p99 < 8 s (NFR-PERF-008)

---

### 4.7 Order Management

#### FR-ORDER-001 — Order Lifecycle (Customer View)
The system shall display the full order lifecycle to the customer.

**Acceptance criteria:**
- Customer can view all their orders (paginated, cursor-based)
- Each order displays: order ID, placed date, items, vendor(s),
  total amount, current status, shipping address
- Order status values visible to customer:
  `PENDING → CONFIRMED → SHIPPED → DELIVERED`
  or terminal failure states: `PAYMENT_FAILED`, `UNDER_REVIEW`
- Customer receives real-time WebSocket push when order status changes
- Customer can view tracking number once vendor marks order as SHIPPED

#### FR-ORDER-002 — Order Lifecycle (Vendor View)
The system shall display vendor-specific sub-orders in the vendor portal.

**Acceptance criteria:**
- Vendor sees only their own sub-orders — never another vendor's
- Each sub-order displays: customer name (first name only for privacy),
  shipping address, line items with quantities, sub-order total,
  current fulfillment status
- Vendor can transition sub-order from CONFIRMED to SHIPPED by
  entering a carrier name and tracking number
- Vendor cannot modify order items or prices
- Vendor receives a notification (email + in-app) when a new
  sub-order is assigned to them

#### FR-ORDER-003 — Order Cancellation
The system shall allow customers to cancel orders that have not yet
been shipped.

**Acceptance criteria:**
- Customer can cancel an order in CONFIRMED status
- Cancellation triggers a saga: inventory released, payment refunded
  to original payment method (simulated by mock gateway)
- Cancellation is not permitted once any sub-order has transitioned
  to SHIPPED
- Cancellation results in: order status → CANCELLED, customer receives
  cancellation confirmation email
- Refund processing time is simulated as 3–5 business days (async)

#### FR-ORDER-004 — Fraud Review Resolution
The system shall allow Admins to resolve orders held under fraud review.

**Acceptance criteria:**
- Admin can view all orders in UNDER_REVIEW status
- Admin can approve the order — triggers manual payment capture via
  Payment service, inventory reservation committed, order transitions
  to CONFIRMED
- Admin can reject the order — reservation released, payment voided,
  order transitions to CANCELLED, customer notified

---

### 4.8 Reviews and Ratings

#### FR-REVIEW-001 — Product Review
The system shall allow customers to review products they have purchased.

**Acceptance criteria:**
- Customer can submit a review only for products in a DELIVERED order
- One review per customer per product (subsequent submission updates
  the existing review)
- Review contains: star rating (1–5), title (optional, max 100 chars),
  body (optional, max 2000 chars)
- Review is stored with: customerId, productId, orderId, rating,
  title, body, submittedAt
- Product's aggregate rating (average + count) is recomputed on every
  review submission
- Reviews are displayed on the product detail page, newest first,
  cursor-paginated

#### FR-REVIEW-002 — Vendor Review
The system shall allow customers to review vendors they have transacted
with.

**Acceptance criteria:**
- Customer can submit one vendor review per completed order containing
  that vendor's products
- Review contains: star rating (1–5), body (optional, max 1000 chars)
- Vendor's aggregate rating is recomputed on every vendor review submission
- Vendor's aggregate rating is displayed on the vendor store page and
  used as a search/filter signal

#### FR-REVIEW-003 — Review Integrity
The system shall enforce that reviews are based on real transactions.

**Acceptance criteria:**
- The system verifies the order is in DELIVERED status before accepting
  a review
- The system verifies the product/vendor being reviewed is present in
  that order
- Review submission endpoint requires a valid JWT with CUSTOMER role
- Attempts to review without a qualifying order return 403 Forbidden

---

### 4.9 Notifications

#### FR-NOTIF-001 — Real-Time Order Updates (WebSocket)
The system shall push real-time order status updates to logged-in
customers via WebSocket.

**Acceptance criteria:**
- Customer browser establishes a Socket.IO connection on login
- Customer is subscribed to a private room scoped to their userId
- On every order status change, the Notification service pushes an
  event to the customer's room
- Event payload includes: orderId, newStatus, timestamp, message
- If the customer is not connected (browser closed), the event is not
  persisted — email is the fallback channel
- WebSocket push is non-blocking — order saga completes regardless of
  whether the customer is connected (NFR-AVAIL-013)

#### FR-NOTIF-002 — Transactional Email
The system shall send transactional emails for the following events.

**Acceptance criteria:**
The following events trigger outbound email:

| Event                        | Recipient | Subject Template                     |
|------------------------------|-----------|--------------------------------------|
| Customer registration        | Customer  | "Verify your SmartCart account"      |
| Vendor application received  | Vendor    | "Your application is under review"   |
| Vendor approved              | Vendor    | "Your vendor account is approved"    |
| Vendor rejected              | Vendor    | "Regarding your vendor application"  |
| Order confirmed              | Customer  | "Order #{{orderId}} confirmed"       |
| Order payment failed         | Customer  | "Payment issue with your order"      |
| Order under review           | Customer  | "Your order is under review"         |
| Sub-order assigned           | Vendor    | "New order to fulfill: #{{orderId}}" |
| Order shipped                | Customer  | "Your order has shipped"             |
| Order delivered              | Customer  | "Your order has been delivered"      |
| Order cancelled              | Customer  | "Order #{{orderId}} cancelled"       |

- Email is sent via Mailhog in local and staging environments
- Email delivery failure does not fail the triggering business
  operation — notifications are best-effort (NFR-AVAIL-013)
- All emails contain the SmartCart brand name, relevant order/account
  details, and a link to the relevant page in the platform

---

### 4.10 Admin Operations

#### FR-ADMIN-001 — Platform Dashboard
The system shall provide Admins with a real-time operational dashboard.

**Acceptance criteria:**
- Dashboard displays the following metrics, refreshed every 60 seconds:
  - Orders per minute (current)
  - Gross Merchandise Value per minute (INR)
  - Active users (currently connected WebSocket sessions)
  - Failed saga count (last 24 hours)
  - DLQ depth across all topics
  - Fraud holds awaiting review
- All metrics are sourced from Prometheus/Grafana — not computed
  on-demand from the database

#### FR-ADMIN-002 — Vendor Management
The system shall allow Admins to manage vendor accounts post-approval.

**Acceptance criteria:**
- Admin can view all vendors filtered by status
- Admin can suspend an APPROVED vendor — their products become INACTIVE
  immediately, new orders cannot be placed against their products
- Admin can reinstate a suspended vendor
- Admin cannot delete a vendor account — historical order integrity
  must be preserved

#### FR-ADMIN-003 — Category Management
The system shall allow Admins to manage the product category taxonomy.

**Acceptance criteria:**
- Admin can create a new category at any level of the hierarchy
- Admin can rename a category
- Admin can deactivate a category (soft delete only)
- Category changes do not delete existing products — affected products
  are flagged for vendor re-categorisation

#### FR-ADMIN-004 — Fraud Review Queue
The system shall provide Admins with a queue for reviewing held orders.

**Acceptance criteria:**
- Queue displays all orders in UNDER_REVIEW status with:
  orderId, customer name, order total, flagged reason from fraud
  detection service, placed timestamp
- Admin can approve or reject each held order (FR-ORDER-004)
- Queue is sorted by oldest first (longest waiting)

---

### 4.11 Analytics

#### FR-ANALYTICS-001 — Vendor Analytics
The system shall provide vendors with their own performance analytics.

**Acceptance criteria:**
- Vendor can view their own analytics dashboard covering:
  - Total revenue (INR) by day/week/month
  - Order volume by day/week/month
  - Top-selling products by revenue and by unit volume
  - Average product rating trend
  - Return rate (orders cancelled before shipment)
- Data is sourced from ClickHouse OLAP store
- Reports cover selectable time ranges: last 7 days, 30 days, 90 days
- Analytics are read-only — no interaction

#### FR-ANALYTICS-002 — Admin Analytics
The system shall provide Admins with platform-level analytics.

**Acceptance criteria:**
- Admin can view:
  - Total platform GMV by day/week/month
  - New customer registrations by day/week/month
  - New vendor applications and approvals by week/month
  - Top vendors by GMV
  - Top products by revenue
  - Fraud hold rate (percentage of orders flagged)

---

### 4.12 AI — Fraud Detection

#### FR-FRAUD-001 — Async Fraud Scoring
The system shall asynchronously score every order for fraud risk.

**Acceptance criteria:**
- After an order is placed, a fraud scoring event is published to Kafka
- Fraud Detection service consumes the event, scores the order, and
  publishes the result
- Scoring considers: order total relative to customer history, velocity
  (orders per hour from this customer), address mismatch signals
- Score is stored against the order
- If score exceeds the HIGH threshold: order transitions to UNDER_REVIEW
  status (triggering FR-ORDER-004 workflow) — regardless of payment
  outcome
- If score is LOW or MEDIUM: order proceeds normally
- Fraud scoring completes within 10 seconds of order placement
- Fraud scoring failure is non-blocking — if the service is down,
  orders proceed normally (fail-open, T3 tier)

---

## 5. Out of Scope

The following are **explicitly excluded** from SmartCart v1. Any
implementation of these requires a new PRD revision.

| Feature                        | Rationale for exclusion                                         |
|--------------------------------|-----------------------------------------------------------------|
| Guest checkout                 | All transactions require authentication; simplifies auth model |
| Multi-currency                 | Single currency (INR) throughout                               |
| Returns and refunds            | Reverse saga complexity; deferred to v2                        |
| Vendor responses to reviews    | Moderation workflow not justified by learning goals            |
| Real payment gateway           | Mock gateway is sufficient; real gateway is an integration, not architecture |
| SMS notifications              | Same pattern as email, no incremental learning value           |
| Courier API integration        | Tracking number is a string field; SmartCart is not a logistics platform |
| Subscription / recurring orders| Out of domain for a learning project                           |
| Wishlists / saved items        | No architectural significance; deferred                        |
| Promotions, coupons, discounts | Pricing engine complexity out of scope                         |
| Multi-language / i18n          | UI in English only                                             |
| Mobile native apps             | Web only (responsive frontend)                                 |
| Seller advertising / promoted listings | Business model feature; not in scope |
| Auction / bidding              | Different pricing model entirely                               |
| B2B / wholesale pricing        | B2C only                                                       |

---

## 6. User Stories — Priority Reference

The following stories represent the minimum viable journeys that must
work end-to-end before the platform is considered functionally complete.
Each story maps to functional requirements defined above.

### Customer Stories (Must Have)

| ID    | Story                                                                        | Maps to               |
|-------|------------------------------------------------------------------------------|-----------------------|
| US-C1 | As a customer, I can register and verify my email to create an account      | FR-AUTH-001           |
| US-C2 | As a customer, I can log in and receive a JWT access token                  | FR-AUTH-003           |
| US-C3 | As a customer, I can search for products by keyword and filter results      | FR-DISC-001           |
| US-C4 | As a customer, I can search semantically and find products by intent        | FR-DISC-002           |
| US-C5 | As a customer, I can upload an image and find visually similar products     | FR-DISC-003           |
| US-C6 | As a customer, I see personalised recommendations on my homepage            | FR-DISC-004           |
| US-C7 | As a customer, I can ask the AI chatbot about products and my orders        | FR-DISC-005           |
| US-C8 | As a customer, I can view a product's details, variants, and stock status   | FR-DISC-006           |
| US-C9 | As a customer, I can add a product variant to my cart                       | FR-CART-001           |
| US-C10| As a customer, I can modify and clear my cart                               | FR-CART-002           |
| US-C11| As a customer, I can check out my cart and receive order confirmation       | FR-CHECKOUT-001/002/003 |
| US-C12| As a customer, I see real-time order status updates in my browser           | FR-NOTIF-001          |
| US-C13| As a customer, I receive email confirmations for all key order events       | FR-NOTIF-002          |
| US-C14| As a customer, I can view my order history and track individual orders      | FR-ORDER-001          |
| US-C15| As a customer, I can cancel a confirmed order before it ships               | FR-ORDER-003          |
| US-C16| As a customer, I can review a product I have purchased                      | FR-REVIEW-001         |
| US-C17| As a customer, I can review a vendor I have transacted with                 | FR-REVIEW-002         |

### Vendor Stories (Must Have)

| ID    | Story                                                                        | Maps to               |
|-------|------------------------------------------------------------------------------|-----------------------|
| US-V1 | As a vendor, I can apply for a vendor account                               | FR-AUTH-002           |
| US-V2 | As a vendor, I receive an email when my application is approved or rejected | FR-NOTIF-002          |
| US-V3 | As a vendor, I can manage my store profile and logo                         | FR-VENDOR-002         |
| US-V4 | As a vendor, I can create product listings with variants                    | FR-CATALOG-001/002    |
| US-V5 | As a vendor, I can publish and unpublish my products                        | FR-CATALOG-003        |
| US-V6 | As a vendor, I can view and fulfill my sub-orders                           | FR-ORDER-002          |
| US-V7 | As a vendor, I can view my sales and performance analytics                  | FR-ANALYTICS-001      |

### Admin Stories (Must Have)

| ID    | Story                                                                        | Maps to               |
|-------|------------------------------------------------------------------------------|-----------------------|
| US-A1 | As an admin, I can review and approve or reject vendor applications         | FR-VENDOR-001         |
| US-A2 | As an admin, I can suspend and reinstate vendor accounts                    | FR-ADMIN-002          |
| US-A3 | As an admin, I can manage the product category taxonomy                     | FR-ADMIN-003          |
| US-A4 | As an admin, I can view platform health metrics on a dashboard              | FR-ADMIN-001          |
| US-A5 | As an admin, I can review and resolve fraud-held orders                     | FR-ADMIN-004          |
| US-A6 | As an admin, I can view platform-level analytics                            | FR-ANALYTICS-002      |

---

## 7. Assumptions

These assumptions are baked into the requirements above. If any changes,
affected requirements must be re-evaluated.

| ID    | Assumption                                                                 |
|-------|----------------------------------------------------------------------------|
| A-001 | All monetary values are in Indian Rupees (INR). No currency conversion.   |
| A-002 | SmartCart operates in a single region (no geo-distribution in scope).      |
| A-003 | All users have a stable internet connection (no offline mode).             |
| A-004 | Tax calculation (GST) is out of scope — prices are displayed inclusive.    |
| A-005 | The mock payment gateway is a SmartCart-owned internal service.            |
| A-006 | Email delivery uses Mailhog in local/staging; no real SMTP credentials.    |
| A-007 | Product images and vendor logos are served directly from MinIO URLs.       |
| A-008 | SmartCart is the sole administrator of the platform — no multi-admin tiers.|
| A-009 | Fraud scoring is advisory — only HIGH scores auto-hold; no ML retraining   |
|       | pipeline is in scope for fraud (rules-based + simple model only).          |

---

## 8. Constraints

These are hard constraints that every implementation decision must respect.

| ID    | Constraint                                                                  |
|-------|-----------------------------------------------------------------------------|
| C-001 | Full local stack must run within 12 GB RAM on a 16 GB developer laptop.    |
| C-002 | Java services cold start < 30 s; Node/Python services < 10 s.              |
| C-003 | No secrets may be committed to version control under any circumstances.     |
| C-004 | Every service must be runnable in isolation via docker compose.             |
| C-005 | Platform must sustain 100 RPS with burst to 300 RPS.                       |

---

## 9. Glossary

| Term              | Definition                                                         |
|-------------------|--------------------------------------------------------------------|
| Sub-order         | The portion of a customer order assigned to a single vendor.       |
| Reservation       | A hold on inventory stock created at checkout, before payment.     |
| Saga              | A distributed transaction coordinated via events with explicit compensation steps. |
| Compensation      | A business-level undo operation that reverses a saga step.         |
| Outbox            | A pattern ensuring events are reliably published by storing them transactionally alongside business data. |
| CQRS              | Command Query Responsibility Segregation — separate models for writes and reads. |
| Idempotency key   | A client-generated token that makes repeated identical requests safe. |
| JTI               | JWT ID — a unique identifier per token used for revocation.        |
| GMV               | Gross Merchandise Value — total value of all orders placed.        |
| DLQ               | Dead Letter Queue — Kafka topic receiving unprocessable messages.  |
| HMAC              | Hash-based Message Authentication Code — used to verify webhook authenticity. |
| RAG               | Retrieval-Augmented Generation — LLM response grounded in retrieved documents. |
| Variant           | A specific purchasable form of a product, defined by attribute-value pairs (e.g. size=M, color=Red). |
| SKU               | Stock Keeping Unit — unique identifier for a specific product variant. |
| Tier              | Service criticality classification (T1/T2/T3) determining SLO rigor. |

---

## 10. Requirement Traceability Matrix

Every functional requirement must eventually trace to at least one of:
an ADR, an OpenAPI spec, an AsyncAPI schema, a service implementation,
and a test case. This matrix is populated incrementally as artifacts
are produced.

| FR ID              | ADR            | OpenAPI / AsyncAPI     | Service(s)                        | Test Coverage |
|--------------------|----------------|------------------------|-----------------------------------|---------------|
| FR-AUTH-001–006    | ADR-002, ADR-005 | auth-service.yaml    | Auth                              | TBD           |
| FR-VENDOR-001–003  | ADR-002        | product-catalog.yaml   | Product Catalog, Auth             | TBD           |
| FR-CATALOG-001–006 | ADR-002        | product-catalog.yaml   | Product Catalog, Inventory        | TBD           |
| FR-DISC-001        | ADR-003        | search-service.yaml    | Search                            | TBD           |
| FR-DISC-002        | ADR-003        | search-service.yaml    | Search, Recommendation            | TBD           |
| FR-DISC-003        | ADR-003        | search-service.yaml    | Search                            | TBD           |
| FR-DISC-004        | ADR-003        | recommendation.yaml    | Recommendation                    | TBD           |
| FR-DISC-005        | ADR-003        | chatbot.yaml           | Chatbot, Order, Product Catalog   | TBD           |
| FR-DISC-006        | ADR-002        | product-catalog.yaml   | Product Catalog, Review           | TBD           |
| FR-CART-001–003    | ADR-002, ADR-003 | cart-service.yaml    | Cart, Inventory                   | TBD           |
| FR-CHECKOUT-001–004| ADR-005        | order-service.yaml     | Order, Inventory, Payment         | TBD           |
| FR-ORDER-001–004   | ADR-005        | order-service.yaml     | Order, Notification               | TBD           |
| FR-REVIEW-001–003  | ADR-002        | review-service.yaml    | Review, Order                     | TBD           |
| FR-NOTIF-001–002   | ADR-003        | asyncapi-notification.yaml | Notification                  | TBD           |
| FR-ADMIN-001–004   | ADR-002        | admin endpoints        | All services                      | TBD           |
| FR-ANALYTICS-001–002| ADR-002      | analytics-service.yaml | Analytics                         | TBD           |
| FR-FRAUD-001       | ADR-005        | asyncapi-fraud.yaml    | Fraud Detection, Order            | TBD           |

---

*End of PRD-001 v1.0.0*
