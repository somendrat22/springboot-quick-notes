# Module 8: REST API Best Practices

---

## 8.1 What Makes a "Good" API?

### Layman Example: A Well-Designed Menu 🍽️

Imagine two restaurants:

**Restaurant A (Bad API):**
```
Menu:
  - getFoodItem
  - doCreateOrder
  - removeItemFromOrder123
  - FOOD_LIST_all_items_fetch
  - process?action=delete&type=order&id=5

→ Confusing, inconsistent, nobody knows how to order.
```

**Restaurant B (Good API):**
```
Menu:
  Food:
    - View all dishes      → GET    /menu/dishes
    - View one dish        → GET    /menu/dishes/42
    - Add a new dish       → POST   /menu/dishes
    - Update a dish        → PUT    /menu/dishes/42
    - Remove a dish        → DELETE /menu/dishes/42

  Orders:
    - View all orders      → GET    /orders
    - Place an order       → POST   /orders
    - View order status    → GET    /orders/7

→ Clean, consistent, self-explanatory.
```

A good REST API is like Restaurant B — **predictable, consistent, and easy to understand**.

---

## 8.2 URL Naming Conventions

### Rules

| Rule | Good ✅ | Bad ❌ |
|------|---------|--------|
| Use **nouns**, not verbs | `/orders` | `/getOrders`, `/createOrder` |
| Use **plural** nouns | `/orders`, `/employees` | `/order`, `/employee` |
| Use **lowercase** | `/api/orders` | `/api/Orders`, `/API/ORDERS` |
| Use **hyphens** for multi-word | `/order-items` | `/orderItems`, `/order_items` |
| **Nest** for relationships | `/departments/5/employees` | `/getEmployeesByDepartment?id=5` |
| Don't include file extensions | `/api/orders` | `/api/orders.json` |
| Don't use trailing slashes | `/api/orders` | `/api/orders/` |

### URL Design Examples

```
# ─── Basic CRUD ──────────────────────────────────────────
GET     /api/orders              → List all orders
GET     /api/orders/42           → Get order #42
POST    /api/orders              → Create a new order
PUT     /api/orders/42           → Replace order #42
PATCH   /api/orders/42           → Partially update order #42
DELETE  /api/orders/42           → Delete order #42

# ─── Nested Resources ───────────────────────────────────
GET     /api/orders/42/items     → Get all items in order #42
POST    /api/orders/42/items     → Add an item to order #42
GET     /api/departments/5/employees  → Get employees in department #5

# ─── Filtering, Sorting, Pagination (use query params) ──
GET     /api/orders?status=pending              → Filter by status
GET     /api/orders?sort=createdAt,desc         → Sort by date descending
GET     /api/orders?page=0&size=20              → Page 0, 20 per page
GET     /api/orders?status=pending&sort=amount,desc&page=0&size=10  → Combined

# ─── Search ─────────────────────────────────────────────
GET     /api/employees/search?q=john            → Search by keyword
GET     /api/products?category=electronics&minPrice=100&maxPrice=500

# ─── Actions (when CRUD doesn't fit) ────────────────────
POST    /api/orders/42/cancel                   → Cancel an order (action)
POST    /api/orders/42/refund                   → Refund an order (action)
POST    /api/accounts/42/activate               → Activate an account
```

---

## 8.3 HTTP Status Codes — Use Them Correctly

### Success (2xx)

| Code | When to Use | Example |
|------|------------|---------|
| `200 OK` | Request succeeded (GET, PUT, PATCH) | `GET /orders/42` → returns the order |
| `201 Created` | New resource created (POST) | `POST /orders` → returns the new order |
| `204 No Content` | Success, nothing to return (DELETE) | `DELETE /orders/42` → empty body |

### Client Errors (4xx)

| Code | When to Use | Example |
|------|------------|---------|
| `400 Bad Request` | Invalid input / validation failure | Missing required field, wrong format |
| `401 Unauthorized` | Not authenticated (who are you?) | Missing or expired token |
| `403 Forbidden` | Authenticated but not authorized (you can't do this) | Regular user trying to access admin endpoint |
| `404 Not Found` | Resource doesn't exist | `GET /orders/99999` |
| `405 Method Not Allowed` | HTTP method not supported | `DELETE /api/login` |
| `409 Conflict` | Conflict with current state | Duplicate email registration |
| `422 Unprocessable Entity` | Valid format but semantically wrong | Order amount is -50 |
| `429 Too Many Requests` | Rate limit exceeded | Too many API calls |

### Server Errors (5xx)

| Code | When to Use |
|------|------------|
| `500 Internal Server Error` | Unhandled exception (bug in your code) |
| `502 Bad Gateway` | Upstream service failed |
| `503 Service Unavailable` | Server overloaded or in maintenance |

---

## 8.4 Standard Error Response Format

Every error should return a **consistent JSON structure**:

```json
{
    "timestamp": "2026-02-24T22:00:00",
    "status": 404,
    "error": "Not Found",
    "message": "Order not found with id: 42",
    "path": "/api/orders/42"
}
```

For validation errors, include field details:

```json
{
    "timestamp": "2026-02-24T22:00:00",
    "status": 400,
    "error": "Validation Failed",
    "message": "One or more fields have invalid values",
    "path": "/api/orders",
    "errors": [
        { "field": "customerName", "message": "must not be blank" },
        { "field": "amount", "message": "must be greater than 0" }
    ]
}
```

**Never return:**
```json
{
    "error": "java.lang.NullPointerException at com.example.service.OrderService.getOrder(OrderService.java:42)"
}
```
Stack traces expose internals — security risk!

---

## 8.5 API Versioning

APIs evolve. You need a strategy to avoid breaking existing clients.

### Layman Analogy: Phone Chargers 🔌

```
iPhone 4:  30-pin connector (v1)
iPhone 5:  Lightning connector (v2)
iPhone 15: USB-C connector (v3)

Apple didn't make the old chargers stop working overnight.
They maintained both for a transition period.
```

### Versioning Strategies

#### Strategy 1: URL Path Versioning (Most Common)

```
GET /api/v1/orders          → Version 1 (returns basic fields)
GET /api/v2/orders          → Version 2 (returns extra fields + pagination metadata)
```

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderControllerV1 {
    // Original implementation
}

@RestController
@RequestMapping("/api/v2/orders")
public class OrderControllerV2 {
    // Enhanced implementation
}
```

#### Strategy 2: Request Header Versioning

```
GET /api/orders
Header: X-API-Version: 1

GET /api/orders
Header: X-API-Version: 2
```

#### Strategy 3: Query Parameter Versioning

```
GET /api/orders?version=1
GET /api/orders?version=2
```

### Which to Use?

| Strategy | Pros | Cons |
|----------|------|------|
| **URL Path** (recommended) | Simple, visible, cacheable | URL changes |
| **Header** | Clean URLs | Hidden, harder to test in browser |
| **Query Param** | Easy to implement | Pollutes query string |

**Most teams use URL path versioning** — it's the most explicit and widely used.

---

## 8.6 Request & Response Best Practices

### Use Envelope Pattern for List Responses

```json
// ❌ BAD — raw array (can't add metadata later)
[
    { "id": 1, "name": "Laptop" },
    { "id": 2, "name": "Phone" }
]

// ✅ GOOD — wrapped with metadata
{
    "data": [
        { "id": 1, "name": "Laptop" },
        { "id": 2, "name": "Phone" }
    ],
    "pagination": {
        "page": 0,
        "size": 10,
        "totalElements": 150,
        "totalPages": 15
    }
}
```

### Use Consistent Naming — camelCase for JSON

```json
// ✅ GOOD — camelCase (Java/JavaScript standard)
{
    "firstName": "John",
    "lastName": "Doe",
    "orderDate": "2026-02-24"
}

// ❌ BAD — inconsistent
{
    "first_name": "John",     // snake_case
    "LastName": "Doe",        // PascalCase
    "order-date": "2026-02-24" // kebab-case
}
```

### Use ISO 8601 for Dates

```json
// ✅ GOOD
{ "createdAt": "2026-02-24T22:00:00Z" }

// ❌ BAD
{ "createdAt": "02/24/2026" }
{ "createdAt": 1740441600000 }
```

---

## 8.7 Pagination Best Practices

```java
// Controller
@GetMapping
public ResponseEntity<Page<OrderResponse>> getOrders(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,      // Don't let clients request 10,000
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "desc") String direction) {

    // CAP the page size to prevent abuse
    int cappedSize = Math.min(size, 100);

    Pageable pageable = PageRequest.of(page, cappedSize,
            Sort.by(Sort.Direction.fromString(direction), sortBy));

    return ResponseEntity.ok(orderService.getOrders(pageable));
}
```

**Key Rules:**
- Always provide **default values** for page, size, sort
- **Cap the page size** (e.g., max 100) to prevent clients from fetching the entire DB
- Return **total count** and **total pages** in the response
- Use **0-based indexing** for pages (Spring default)

---

## 8.8 HATEOAS (Hypermedia) — Briefly

### Layman Analogy: GPS Navigation 🗺️

```
Without HATEOAS:
  You arrive at a city → you have to memorize all the street names in advance.

With HATEOAS:
  You arrive at a city → every intersection has signboards telling you where you can go next.
```

HATEOAS means the API response includes **links** to related actions:

```json
{
    "id": 42,
    "customerName": "John",
    "status": "PENDING",
    "_links": {
        "self":    { "href": "/api/orders/42" },
        "cancel":  { "href": "/api/orders/42/cancel" },
        "items":   { "href": "/api/orders/42/items" },
        "customer": { "href": "/api/customers/7" }
    }
}
```

**In practice:** HATEOAS is theoretically ideal but rarely fully implemented. Know the concept for interviews but don't stress about implementing it.

---

## 8.9 Security Best Practices

| Practice | Why |
|----------|-----|
| Always use **HTTPS** | Prevents eavesdropping |
| **Never** expose stack traces | Security risk — reveals internals |
| Validate **all** input | Prevent SQL injection, XSS |
| Use **rate limiting** | Prevent abuse / DDoS |
| Authenticate with **tokens** (JWT) | Stateless, scalable |
| Implement **CORS** properly | Control which domains can call your API |
| Don't expose **sensitive data** | Use DTOs, never return passwords or internal IDs |
| Use **pagination** | Prevent clients from dumping your entire database |

---

## 8.10 Idempotency

### Layman Analogy: Elevator Button 🛗

```
You press the elevator button once  → elevator comes.
You press it 10 more times         → elevator still comes once (same result).
That button is IDEMPOTENT.

You place an order once → 1 order created.
You accidentally click "Place Order" 3 more times → 4 orders created! 😱
That button is NOT idempotent.
```

### HTTP Methods & Idempotency

| Method | Idempotent? | Explanation |
|--------|-------------|-------------|
| GET | Yes | Reading data doesn't change anything |
| PUT | Yes | Replacing with same data → same result every time |
| DELETE | Yes | Deleting something twice → it's still deleted |
| PATCH | Depends | Usually not guaranteed |
| POST | **No** | Creating a resource twice → two resources |

### Making POST Idempotent

Use an **idempotency key**:

```
POST /api/orders
Header: Idempotency-Key: abc-123-xyz

First call  → creates order, stores key "abc-123-xyz" → returns 201
Second call → detects key "abc-123-xyz" already processed → returns cached 201 response
```

---

## 8.11 Common Interview Questions & Answers

**Q: What is the difference between PUT and PATCH?**
> **PUT** replaces the **entire** resource — you must send ALL fields. **PATCH** updates **partially** — send only the fields you want to change. PUT is idempotent; PATCH may not be.

**Q: Why use plural nouns in URLs?**
> `/orders` represents a **collection**. `/orders/42` represents a single item in that collection. It reads naturally: "get me the list of orders" vs "get me order 42 from the orders collection."

**Q: How do you handle API versioning?**
> URL path versioning (`/api/v1/orders`, `/api/v2/orders`) is the most common and explicit approach. It's simple, visible, and cacheable. Support old versions for a deprecation period before sunsetting them.

**Q: How do you secure a REST API?**
> Use HTTPS, JWT tokens for authentication, role-based authorization, input validation, rate limiting, CORS configuration, and never expose internal errors or sensitive data.

**Q: What is idempotency and why does it matter?**
> An operation is idempotent if calling it multiple times produces the same result as calling it once. It matters because networks are unreliable — a client might retry a request that already succeeded. GET, PUT, and DELETE are idempotent; POST is not (unless you use idempotency keys).

---

## 8.12 Quick Revision Table

| Concept | One-Liner |
|---------|-----------|
| URL naming | Plural nouns, lowercase, hyphens, no verbs |
| HTTP methods | GET=read, POST=create, PUT=replace, PATCH=update, DELETE=remove |
| Status codes | 2xx=success, 4xx=client error, 5xx=server error |
| Versioning | `/api/v1/...` — most common approach |
| Pagination | Always paginate lists, cap page size, return total count |
| Error format | Consistent JSON with timestamp, status, message, path |
| Idempotency | Same request N times = same result (GET, PUT, DELETE are idempotent) |
| HATEOAS | Response includes links to related actions |
| Security | HTTPS, JWT, validation, rate limiting, DTOs, no stack traces |

---

### Interview Tip 🗣️

> "A good REST API is **predictable and consistent**. Use plural nouns for URLs, proper HTTP methods and status codes, paginate all lists, version your API with URL paths, and always return a standard error format. Security-wise: HTTPS everywhere, JWT for auth, validate all input, and never expose internals."

---

[⬅️ Back to Curriculum](./README.md) | [⬅️ Previous: Annotations](./07-spring-boot-annotations.md) | [Next: Spring Security ➡️](./09-spring-security-basics.md)
