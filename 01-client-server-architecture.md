# Module 1: Client-Server Architecture

---

## 1.1 What is Client-Server Architecture?

It's a model where two parties communicate over a network:
- **Client** — the one who **requests** something.
- **Server** — the one who **responds** with something.

That's it. Every website you visit, every app you use, every API call — all follow this pattern.

---

## 1.2 Layman Example: The Restaurant 🍽️

Imagine you walk into a restaurant:

| Restaurant | Tech World |
|------------|-----------|
| **You** (the customer) sit at a table and place an order | **Client** (browser, mobile app, Postman) sends an HTTP request |
| **The waiter** carries your order to the kitchen | **The Internet / Network** carries the request to the server |
| **The kitchen** reads the order, cooks the food | **The Server** receives the request, processes it (runs business logic, queries DB) |
| The waiter brings food back to your table | The **HTTP Response** (JSON, HTML, file) is sent back to the client |
| You eat the food | The client **renders/displays** the response |

### Key Observations from the Analogy

1. **You don't go into the kitchen** — the client never directly accesses the server's internals (database, file system, etc.).
2. **The waiter speaks a common language** — both you and the kitchen understand "menu items." In tech, this common language is **HTTP**.
3. **Multiple customers, one kitchen** — many clients can talk to the same server simultaneously.
4. **The kitchen doesn't come to you** — the server only responds when asked. It doesn't push food to your table randomly (unless you're using WebSockets, which is a different story).

---

## 1.3 The Players in Detail

### Client

The client is anything that **initiates** a request:

| Client Type | Example |
|-------------|---------|
| Web Browser | Chrome, Firefox, Safari |
| Mobile App | Instagram app, Uber app |
| Desktop App | Slack, VS Code (for extensions) |
| CLI Tool | `curl`, `wget`, Postman |
| Another Server | Microservice A calling Microservice B |

**Important:** A "client" is a **role**, not a device. Your laptop can be a client (when using Chrome) and a server (when running a local dev server) at the same time.

### Server

The server is anything that **listens** for requests and **responds**:

| Server Type | Example |
|-------------|---------|
| Web Server | Apache, Nginx |
| Application Server | Tomcat (Spring Boot's embedded server), Node.js |
| Database Server | MySQL, PostgreSQL, MongoDB |
| File Server | AWS S3, FTP server |
| Mail Server | SMTP server |

In Spring Boot, the embedded **Tomcat** server listens on a port (default `8080`) and routes incoming HTTP requests to your controllers.

---

## 1.4 HTTP — The Language They Speak

HTTP (HyperText Transfer Protocol) is the **agreed-upon format** for communication.

### Layman Analogy: Ordering via a Form

Think of HTTP as a **standardized order form** at the restaurant:

```
┌─────────────────────────────────────────────┐
│            CUSTOMER ORDER FORM              │
├─────────────────────────────────────────────┤
│ Action:    □ VIEW  □ CREATE  □ UPDATE  □ DELETE  │  ← HTTP Method
│ Table No:  __42__                            │  ← URL / Endpoint
│ Special Instructions: "No onions"            │  ← Headers
│ Order Details: "1x Burger, 1x Fries"         │  ← Request Body
└─────────────────────────────────────────────┘
```

### HTTP Methods (Verbs)

| Method | Purpose | Restaurant Analogy | Safe? | Idempotent? |
|--------|---------|-------------------|-------|-------------|
| `GET` | Retrieve data | "Can I see the menu?" | Yes | Yes |
| `POST` | Create new data | "I'd like to place a new order" | No | No |
| `PUT` | Replace/Update entire resource | "Replace my entire order with this new one" | No | Yes |
| `PATCH` | Partially update a resource | "Change just the drink in my order to Coke" | No | No |
| `DELETE` | Remove data | "Cancel my order" | No | Yes |

**Safe** = doesn't change anything on the server.
**Idempotent** = calling it 10 times has the same effect as calling it once.

### HTTP Status Codes

Think of these as the **kitchen's reply slip**:

| Code Range | Meaning | Analogy |
|------------|---------|---------|
| **2xx** | Success | "Your food is ready!" |
| `200 OK` | Request succeeded | Order served |
| `201 Created` | New resource created | New order placed successfully |
| `204 No Content` | Success, nothing to return | "Order cancelled, nothing to give back" |
| **3xx** | Redirection | "We moved to a new location, go there" |
| `301 Moved Permanently` | Resource moved | Restaurant relocated permanently |
| `302 Found` | Temporary redirect | "Kitchen is busy, go to the food truck outside" |
| **4xx** | Client Error (YOUR fault) | "You ordered something wrong" |
| `400 Bad Request` | Malformed request | "I can't read your handwriting" |
| `401 Unauthorized` | Not authenticated | "Who are you? Show your reservation" |
| `403 Forbidden` | Authenticated but not allowed | "You have a reservation but this is the VIP section" |
| `404 Not Found` | Resource doesn't exist | "We don't serve that dish" |
| `409 Conflict` | Conflict with current state | "That table is already taken" |
| **5xx** | Server Error (KITCHEN's fault) | "Kitchen caught fire" |
| `500 Internal Server Error` | Generic server error | "Something went wrong in the kitchen" |
| `502 Bad Gateway` | Upstream server failed | "Our supplier didn't deliver ingredients" |
| `503 Service Unavailable` | Server overloaded / maintenance | "Kitchen is closed for cleaning" |

---

## 1.5 The Complete Request-Response Cycle

Here's what happens when you type `https://api.example.com/orders/42` in your browser:

```
Step 1: Browser creates an HTTP GET request
         ┌──────────────────────────────────┐
         │ GET /orders/42 HTTP/1.1          │
         │ Host: api.example.com            │
         │ Accept: application/json         │
         │ Authorization: Bearer eyJhb...   │
         └──────────────────────────────────┘

Step 2: DNS Resolution
         "api.example.com" → DNS Server → "34.120.5.10"
         (Like looking up a restaurant's address in Google Maps)

Step 3: TCP Connection (3-way handshake)
         Client: "Hey, are you there?" (SYN)
         Server: "Yes, I'm here!" (SYN-ACK)
         Client: "Great, let's talk!" (ACK)
         (Like calling ahead to confirm the restaurant is open)

Step 4: TLS Handshake (for HTTPS)
         Client and server agree on encryption keys
         (Like agreeing on a secret code so no one eavesdrops)

Step 5: Server receives the request
         Tomcat (web server) → DispatcherServlet → Your @Controller

Step 6: Application processes the request
         Controller → Service → Repository → Database
         Database → Repository → Service → Controller

Step 7: Server sends HTTP response
         ┌──────────────────────────────────┐
         │ HTTP/1.1 200 OK                  │
         │ Content-Type: application/json   │
         │                                  │
         │ {                                │
         │   "id": 42,                      │
         │   "customer": "John",            │
         │   "product": "Laptop",           │
         │   "amount": 999.99               │
         │ }                                │
         └──────────────────────────────────┘

Step 8: Browser receives & renders the response
```

---

## 1.6 REST — A Style of Building APIs

REST (Representational State Transfer) is not a protocol — it's a **set of conventions** for designing APIs on top of HTTP.

### Layman Analogy: Library System 📚

| REST Concept | Library Analogy |
|-------------|-----------------|
| **Resource** | A book |
| **URI** | The book's shelf number: `/books/978-0-13-468599-1` |
| **HTTP Method** | What you want to do: borrow (GET), donate (POST), replace (PUT), discard (DELETE) |
| **Representation** | The format you get: JSON, XML (like getting the book in English or Hindi) |
| **Stateless** | The librarian doesn't remember you between visits — you show your library card every time |

### RESTful URL Design

```
# Good (noun-based, resource-oriented)
GET    /api/orders          → Get all orders
GET    /api/orders/42       → Get order #42
POST   /api/orders          → Create a new order
PUT    /api/orders/42       → Replace order #42
PATCH  /api/orders/42       → Partially update order #42
DELETE /api/orders/42       → Delete order #42

# Bad (verb-based — this is NOT RESTful)
GET    /api/getOrders
POST   /api/createOrder
POST   /api/deleteOrder/42
```

### Key REST Principles

1. **Stateless** — Each request contains ALL information needed to process it. The server doesn't store session state between requests.
2. **Resource-based** — Everything is a resource identified by a URI.
3. **Uniform Interface** — Use standard HTTP methods (GET, POST, PUT, DELETE).
4. **Client-Server Separation** — Client and server evolve independently.

---

## 1.7 Common Interview Questions & Answers

**Q: What is the difference between a web server and an application server?**
> A web server (Nginx, Apache) serves **static content** (HTML, CSS, images) and can act as a reverse proxy. An application server (Tomcat, JBoss) runs **application logic** (your Java code). Spring Boot embeds Tomcat so you get both in one.

**Q: What does "stateless" mean in REST?**
> The server does NOT remember anything about the client between requests. Every request must carry all necessary info (like an auth token). This makes the server easy to scale — any server instance can handle any request.

**Q: What is the difference between PUT and PATCH?**
> PUT **replaces the entire resource** — you must send all fields. PATCH **partially updates** — you only send the fields you want to change.

**Q: What happens if you send a GET request with a body?**
> Technically HTTP allows it, but it's **non-standard and discouraged**. Most servers and proxies ignore the body of a GET request. Use POST or PUT if you need to send data.

**Q: What is the difference between 401 and 403?**
> `401 Unauthorized` = "I don't know who you are" (not authenticated). `403 Forbidden` = "I know who you are, but you're not allowed" (not authorized).

---

## 1.8 Quick Revision Table

| Concept | One-Liner |
|---------|-----------|
| Client | The one who asks (browser, app, curl) |
| Server | The one who answers (Tomcat, Nginx) |
| HTTP | The language they speak |
| GET | Read data |
| POST | Create data |
| PUT | Replace data |
| DELETE | Remove data |
| 2xx | Success |
| 4xx | Client messed up |
| 5xx | Server messed up |
| REST | A convention for designing clean APIs over HTTP |
| Stateless | Server doesn't remember you — send everything each time |

---

### Interview Tip 🗣️

> "In a client-server model, the **client is dumb** — it only knows how to ask. The **server is smart** — it knows how to answer. They communicate using a shared language called HTTP, following conventions called REST."

---

[⬅️ Back to Curriculum](./README.md) | [Next: Dependency Injection ➡️](./02-dependency-injection-ioc.md)
