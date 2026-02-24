# Module 3: Controller → Service → Repository (Spring Boot Layered Architecture)

---

## 3.1 What is Layered Architecture?

It's a way of organizing your code into **layers**, where each layer has **one specific job** and only talks to the layer directly below it.

```
    ┌─────────────────────────────┐
    │         CLIENT              │  (Browser, Mobile App, Postman)
    └─────────────┬───────────────┘
                  │ HTTP Request
                  ▼
    ┌─────────────────────────────┐
    │       CONTROLLER LAYER      │  Handles HTTP in/out
    └─────────────┬───────────────┘
                  │ Java method call
                  ▼
    ┌─────────────────────────────┐
    │        SERVICE LAYER        │  Business logic
    └─────────────┬───────────────┘
                  │ Java method call
                  ▼
    ┌─────────────────────────────┐
    │      REPOSITORY LAYER       │  Data access
    └─────────────┬───────────────┘
                  │ SQL / JPQL
                  ▼
    ┌─────────────────────────────┐
    │         DATABASE            │  MySQL, PostgreSQL, MongoDB...
    └─────────────────────────────┘
```

---

## 3.2 Layman Example: A Fast-Food Restaurant 🍔

Let's map each layer to a role in a McDonald's:

| Layer | McDonald's Role | What They Do | What They DON'T Do |
|-------|----------------|-------------|-------------------|
| **Controller** | **Cashier at the counter** | Takes your order, confirms it, gives you a receipt | Doesn't cook food, doesn't go to the fridge |
| **Service** | **Head Chef / Kitchen Manager** | Reads the order, decides what to prepare, applies recipes (business rules), assembles the combo | Doesn't talk to customers, doesn't go to the warehouse |
| **Repository** | **Pantry / Warehouse Worker** | Gets raw ingredients from the fridge/warehouse, puts things back | Doesn't know recipes, doesn't talk to customers |
| **Database** | **The actual fridge / warehouse** | Stores all the raw ingredients | Just stores stuff, doesn't think |

### The Flow — Customer Orders a Big Mac Combo

```
1. YOU (Client) → walk up to the counter
   "I want a Big Mac Combo, large fries, Coke"

2. CASHIER (Controller) → takes the order
   - Validates: "Is this a valid menu item? Yes."
   - Doesn't cook anything!
   - Passes the order slip to the kitchen

3. HEAD CHEF (Service) → processes the order
   - Business rule: "Big Mac Combo = 1 Big Mac + 1 Large Fries + 1 Large Coke"
   - Business rule: "It's Happy Hour → apply 10% discount"
   - Asks the pantry worker for ingredients

4. PANTRY WORKER (Repository) → fetches ingredients
   - Goes to the fridge (database)
   - Gets buns, patties, lettuce, fries, Coke syrup
   - Hands them to the chef

5. HEAD CHEF (Service) → assembles the food
   - Cooks the patty, assembles the burger, bags everything
   - Hands the bag to the cashier

6. CASHIER (Controller) → gives you the food
   - "Here's your order! That'll be $8.99"
   - Returns the response to the customer
```

### Why Not Skip Layers?

**Q: Why can't the cashier just go to the fridge directly?**

```
❌ BAD: Controller calls Repository directly

    If the cashier goes to the fridge:
    - Who applies the Happy Hour discount? Nobody.
    - Who validates the combo rules? Nobody.
    - Who handles "out of stock" logic? Nobody.

    Business logic gets scattered or lost.
```

```
✅ GOOD: Controller → Service → Repository

    Each person does ONE job:
    - Cashier handles customer interaction
    - Chef handles recipes and rules
    - Pantry handles storage

    Business logic is centralized in the Service layer.
```

---

## 3.3 The Entity — What is Stored in the Database

Before the layers, let's define **what** we're storing.

### Layman Analogy

The Entity is like a **standardized box** in the warehouse. Each box has labels:
- Box ID (primary key)
- Customer Name
- Product
- Amount
- Order Date

### In Code

```java
import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity                              // "This class maps to a database table"
@Table(name = "orders")              // "The table is called 'orders'"
public class Order {

    @Id                              // "This is the primary key"
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // "Auto-increment"
    private Long id;

    @Column(name = "customer_name", nullable = false)  // Maps to column "customer_name"
    private String customerName;

    @Column(nullable = false)
    private String product;

    @Column(nullable = false)
    private Double amount;

    @Column(name = "order_date")
    private LocalDateTime orderDate;

    // Default constructor (required by JPA)
    public Order() {}

    // Parameterized constructor
    public Order(String customerName, String product, Double amount) {
        this.customerName = customerName;
        this.product = product;
        this.amount = amount;
        this.orderDate = LocalDateTime.now();
    }

    // Getters and Setters
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }

    public String getCustomerName() { return customerName; }
    public void setCustomerName(String customerName) { this.customerName = customerName; }

    public String getProduct() { return product; }
    public void setProduct(String product) { this.product = product; }

    public Double getAmount() { return amount; }
    public void setAmount(Double amount) { this.amount = amount; }

    public LocalDateTime getOrderDate() { return orderDate; }
    public void setOrderDate(LocalDateTime orderDate) { this.orderDate = orderDate; }
}
```

### What This Creates in the Database

```sql
CREATE TABLE orders (
    id            BIGINT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(255) NOT NULL,
    product       VARCHAR(255) NOT NULL,
    amount        DOUBLE NOT NULL,
    order_date    DATETIME
);
```

---

## 3.4 The Repository Layer — Talks to the Database

### What It Does

- **ONLY** job: Get data from the database, save data to the database.
- **No business logic here.** Just CRUD operations.
- In Spring Boot, you usually just **extend an interface** — Spring generates the implementation.

### In Code

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    //                                     ↑ Entity   ↑ Primary Key type

    // Spring auto-generates the SQL from the method name!
    List<Order> findByCustomerName(String customerName);

    List<Order> findByProduct(String product);

    List<Order> findByAmountGreaterThan(Double amount);
}
```

**That's it.** No implementation class. No SQL. Spring Data JPA reads the method names and generates everything.

---

## 3.5 The Service Layer — Contains Business Logic

### What It Does

- **The brain** of your application.
- Contains all **business rules**: validation, calculations, orchestration.
- Calls the Repository to get/save data.
- **Never touches HTTP** — doesn't know about requests, responses, status codes.

### In Code

```java
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
public class OrderService {

    private final OrderRepository orderRepository;

    // Constructor injection — Spring injects the repository automatically
    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    // ─── READ OPERATIONS ────────────────────────────────────

    public List<Order> getAllOrders() {
        return orderRepository.findAll();
    }

    public Order getOrderById(Long id) {
        return orderRepository.findById(id)
                .orElseThrow(() -> new RuntimeException("Order not found with id: " + id));
    }

    public List<Order> getOrdersByCustomer(String customerName) {
        return orderRepository.findByCustomerName(customerName);
    }

    // ─── WRITE OPERATIONS ───────────────────────────────────

    @Transactional  // If anything fails, rollback the entire operation
    public Order placeOrder(Order order) {

        // ─── Business Rule 1: Minimum order amount ───
        if (order.getAmount() < 5.0) {
            throw new IllegalArgumentException("Minimum order amount is $5.00");
        }

        // ─── Business Rule 2: Set order date ───
        order.setOrderDate(java.time.LocalDateTime.now());

        // ─── Business Rule 3: Apply discount for large orders ───
        if (order.getAmount() > 100.0) {
            double discountedAmount = order.getAmount() * 0.90;  // 10% off
            order.setAmount(discountedAmount);
        }

        return orderRepository.save(order);
    }

    @Transactional
    public Order updateOrder(Long id, Order updatedOrder) {
        Order existingOrder = getOrderById(id);  // reuse existing method

        existingOrder.setCustomerName(updatedOrder.getCustomerName());
        existingOrder.setProduct(updatedOrder.getProduct());
        existingOrder.setAmount(updatedOrder.getAmount());

        return orderRepository.save(existingOrder);
    }

    @Transactional
    public void deleteOrder(Long id) {
        Order order = getOrderById(id);  // verify it exists first
        orderRepository.delete(order);
    }
}
```

### Key Observations

1. **Business rules live HERE** — minimum amount check, discount logic, date setting.
2. **`@Transactional`** — ensures database operations are atomic (all succeed or all rollback).
3. **No HTTP knowledge** — the service doesn't know about `@GetMapping`, `ResponseEntity`, or status codes.
4. **Reuses its own methods** — `updateOrder` calls `getOrderById` internally.

---

## 3.6 The Controller Layer — The HTTP Entry Point

### What It Does

- **The front door** of your application.
- Receives HTTP requests, **delegates** to the Service layer, and returns HTTP responses.
- **No business logic here.** Just: receive → delegate → respond.
- Handles: URL mapping, request parsing, response formatting, status codes.

### In Code

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@RestController                      // This class handles HTTP requests and returns JSON
@RequestMapping("/api/orders")       // Base URL for all endpoints in this controller
public class OrderController {

    private final OrderService orderService;

    // Constructor injection
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    // ─── GET /api/orders ────────────────────────────────────
    @GetMapping
    public ResponseEntity<List<Order>> getAllOrders() {
        List<Order> orders = orderService.getAllOrders();
        return ResponseEntity.ok(orders);  // 200 OK
    }

    // ─── GET /api/orders/{id} ───────────────────────────────
    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrderById(@PathVariable Long id) {
        Order order = orderService.getOrderById(id);
        return ResponseEntity.ok(order);  // 200 OK
    }

    // ─── GET /api/orders/customer?name=John ─────────────────
    @GetMapping("/customer")
    public ResponseEntity<List<Order>> getOrdersByCustomer(@RequestParam String name) {
        List<Order> orders = orderService.getOrdersByCustomer(name);
        return ResponseEntity.ok(orders);  // 200 OK
    }

    // ─── POST /api/orders ───────────────────────────────────
    @PostMapping
    public ResponseEntity<Order> placeOrder(@RequestBody Order order) {
        Order savedOrder = orderService.placeOrder(order);
        return ResponseEntity.status(HttpStatus.CREATED).body(savedOrder);  // 201 Created
    }

    // ─── PUT /api/orders/{id} ───────────────────────────────
    @PutMapping("/{id}")
    public ResponseEntity<Order> updateOrder(@PathVariable Long id, @RequestBody Order order) {
        Order updatedOrder = orderService.updateOrder(id, order);
        return ResponseEntity.ok(updatedOrder);  // 200 OK
    }

    // ─── DELETE /api/orders/{id} ────────────────────────────
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteOrder(@PathVariable Long id) {
        orderService.deleteOrder(id);
        return ResponseEntity.noContent().build();  // 204 No Content
    }
}
```

### Key Annotations Explained

| Annotation | Purpose | Example |
|-----------|---------|---------|
| `@RestController` | Marks class as a REST API controller (returns JSON, not HTML) | Class-level |
| `@RequestMapping` | Base URL path for all endpoints in this controller | `"/api/orders"` |
| `@GetMapping` | Handles HTTP GET | `GET /api/orders` |
| `@PostMapping` | Handles HTTP POST | `POST /api/orders` |
| `@PutMapping` | Handles HTTP PUT | `PUT /api/orders/42` |
| `@DeleteMapping` | Handles HTTP DELETE | `DELETE /api/orders/42` |
| `@PathVariable` | Extracts value from URL path | `/orders/{id}` → `id = 42` |
| `@RequestParam` | Extracts value from query string | `/orders?name=John` → `name = "John"` |
| `@RequestBody` | Deserializes JSON body into Java object | `{ "product": "Laptop" }` → `Order` |
| `ResponseEntity<T>` | Wrapper that lets you set status code + headers + body | `ResponseEntity.ok(data)` |

### `@PathVariable` vs `@RequestParam`

```
@PathVariable → part of the URL path
    GET /api/orders/42          →  @PathVariable Long id = 42

@RequestParam → part of the query string (after ?)
    GET /api/orders?name=John   →  @RequestParam String name = "John"

When to use which?
    - PathVariable → identifying a SPECIFIC resource (id, slug)
    - RequestParam → filtering, sorting, pagination (optional parameters)
```

---

## 3.7 Connecting Spring Boot to a Database

### application.properties

This file tells Spring Boot **where** the database is and **how** to connect:

```properties
# ─── Database Connection ──────────────────────────────────
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# ─── JPA / Hibernate Settings ────────────────────────────
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
```

### What Each Property Means

| Property | What It Does | Layman Analogy |
|----------|-------------|----------------|
| `datasource.url` | Database address | Restaurant's physical address |
| `datasource.username` | DB login username | Employee badge to enter the kitchen |
| `datasource.password` | DB login password | PIN code for the badge |
| `driver-class-name` | Which DB driver to use | Knowing whether to drive a car or a truck to get there |
| `ddl-auto` | How to handle schema changes | How to rearrange the warehouse |
| `show-sql` | Print SQL queries to console | Showing the chef's recipe steps in real-time |

### `ddl-auto` Options (IMPORTANT for interviews)

| Value | What Happens | When to Use |
|-------|-------------|-------------|
| `none` | Does nothing to the schema | **Production** — you manage schema manually with migrations |
| `validate` | Checks that entities match the DB schema, throws error if not | **Production** — safety check |
| `update` | Adds new columns/tables but never deletes existing ones | **Development** — convenient but risky |
| `create` | Drops all tables and recreates them every time the app starts | **Testing** — fresh start each time |
| `create-drop` | Creates on startup, drops everything on shutdown | **Unit tests** — temporary DB |

### For Different Databases

```properties
# ─── MySQL ────────────────────────────────────
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# ─── PostgreSQL ───────────────────────────────
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.driver-class-name=org.postgresql.Driver

# ─── H2 (In-Memory — great for testing) ──────
spring.datasource.url=jdbc:h2:mem:testdb
spring.datasource.driver-class-name=org.h2.Driver
spring.h2.console.enabled=true
```

### Maven Dependencies (pom.xml)

```xml
<!-- Spring Data JPA -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- MySQL Driver -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- OR H2 for testing -->
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
    <scope>runtime</scope>
</dependency>
```

---

## 3.8 How Spring Boot Wires It All Together (Behind the Scenes)

When your Spring Boot app starts:

```
1. @SpringBootApplication triggers COMPONENT SCANNING
   → Finds @Controller, @Service, @Repository classes

2. Spring IoC container creates beans:
   → OrderRepository (proxy generated by Spring Data JPA)
   → OrderService (with OrderRepository injected)
   → OrderController (with OrderService injected)

3. Embedded Tomcat starts on port 8080

4. DispatcherServlet registers URL mappings:
   → GET  /api/orders      → OrderController.getAllOrders()
   → GET  /api/orders/{id} → OrderController.getOrderById()
   → POST /api/orders      → OrderController.placeOrder()
   ... etc.

5. Request arrives: GET /api/orders/42
   → Tomcat → DispatcherServlet → OrderController.getOrderById(42)
   → OrderController → OrderService.getOrderById(42)
   → OrderService → OrderRepository.findById(42)
   → OrderRepository → SQL: SELECT * FROM orders WHERE id = 42
   → Database returns row → mapped to Order object
   → Flows back up: Repository → Service → Controller → JSON Response
```

---

## 3.9 The Complete Project Structure

```
src/
├── main/
│   ├── java/
│   │   └── com/example/orderapp/
│   │       ├── OrderAppApplication.java          ← Main class (@SpringBootApplication)
│   │       ├── controller/
│   │       │   └── OrderController.java          ← @RestController
│   │       ├── service/
│   │       │   └── OrderService.java             ← @Service
│   │       ├── repository/
│   │       │   └── OrderRepository.java          ← @Repository (interface)
│   │       ├── entity/  (or model/)
│   │       │   └── Order.java                    ← @Entity
│   │       ├── dto/                              ← Data Transfer Objects (optional)
│   │       │   └── OrderDTO.java
│   │       └── exception/                        ← Custom exceptions (optional)
│   │           └── OrderNotFoundException.java
│   └── resources/
│       ├── application.properties                ← DB config, server config
│       └── application-dev.properties            ← Dev-specific config
└── test/
    └── java/
        └── com/example/orderapp/
            ├── controller/
            │   └── OrderControllerTest.java
            └── service/
                └── OrderServiceTest.java
```

---

## 3.10 Common Interview Questions & Answers

**Q: Why do we separate Controller, Service, and Repository?**
> **Separation of Concerns.** Each layer has one job. This makes the code easier to test (mock one layer to test another), maintain (change DB layer without touching business logic), and understand (each file has a clear purpose).

**Q: Can a Controller call a Repository directly?**
> Technically yes, but it's a **bad practice**. You'd bypass business logic, making the controller fat and the code harder to maintain and test. Always go through the Service layer.

**Q: What is `@Transactional` and why is it on the Service layer?**
> `@Transactional` ensures that all database operations within a method are **atomic** — either ALL succeed or ALL are rolled back. It belongs on the Service layer because that's where business operations (which may involve multiple DB calls) live.

**Q: What's the difference between `@Controller` and `@RestController`?**
> `@Controller` returns **view names** (HTML templates, used in MVC with Thymeleaf).
> `@RestController` = `@Controller` + `@ResponseBody` — returns **data** (JSON/XML) directly.

**Q: What is a DTO and why use it?**
> A DTO (Data Transfer Object) is a **separate class used for API input/output** instead of exposing your Entity directly. Reasons:
> - Hide sensitive fields (password, internal IDs)
> - Validate input separately from the entity
> - Decouple API contract from database schema

---

## 3.11 Quick Revision Table

| Concept | One-Liner |
|---------|-----------|
| Controller | Front door — handles HTTP request/response |
| Service | Brain — contains business rules |
| Repository | Bridge to the database — handles CRUD |
| Entity | A Java class that maps to a database table |
| `@RestController` | Returns JSON (not HTML) |
| `@RequestMapping` | Sets the base URL path |
| `@PathVariable` | Extracts value from URL path (`/orders/{id}`) |
| `@RequestParam` | Extracts value from query string (`?name=John`) |
| `@RequestBody` | Deserializes JSON body → Java object |
| `@Transactional` | All-or-nothing database operations |
| `ddl-auto=update` | Auto-sync entity changes to DB schema (dev only) |

---

### Interview Tip 🗣️

> "Controller handles **what comes in and goes out** (HTTP). Service handles **what to do** (business logic). Repository handles **where to get it** (database). They never skip layers — Controller never talks directly to the Repository."

---

[⬅️ Back to Curriculum](./README.md) | [⬅️ Previous: Dependency Injection](./02-dependency-injection-ioc.md) | [Next: JPA Repository ➡️](./04-jpa-repository.md)
