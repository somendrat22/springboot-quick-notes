# Module 11: Real-World Project Scenarios — DB Design, Endpoints & In-Depth Details

---

## Why This Module?

In interviews, you're often asked: *"How would you design the backend for X?"* This module walks through **5 real-world project scenarios** end-to-end:

1. [E-Commerce Platform](#scenario-1-e-commerce-platform)
2. [Blog / Content Management System](#scenario-2-blog--content-management-system)
3. [Employee Management System (HR Portal)](#scenario-3-employee-management-system-hr-portal)
4. [Food Delivery App](#scenario-4-food-delivery-app)
5. [Banking / Wallet System](#scenario-5-banking--wallet-system)

For each scenario, you'll get:
- **Requirements** (what the app should do)
- **Database design** (tables, relationships, ER diagram in text)
- **Complete API endpoints** (RESTful)
- **Key business rules**
- **Entity code** (Spring Boot)
- **Interview talking points**

---

---

# Scenario 1: E-Commerce Platform

## Requirements

```
A simple e-commerce platform where:
- Users can register and log in
- Users can browse products by category
- Users can add products to a cart
- Users can place orders from their cart
- Admins can manage products and categories
- Order history is maintained
```

## Database Design

### ER Diagram (Text)

```
┌──────────┐       ┌──────────────┐       ┌──────────────┐
│  users   │       │  categories  │       │   products   │
├──────────┤       ├──────────────┤       ├──────────────┤
│ id (PK)  │       │ id (PK)      │       │ id (PK)      │
│ name     │       │ name         │       │ name         │
│ email    │       │ description  │       │ description  │
│ password │       │ created_at   │       │ price        │
│ role     │       └──────┬───────┘       │ stock_qty    │
│ phone    │              │ 1             │ image_url    │
│ created_at│             │               │ category_id (FK)──→ categories.id
└────┬─────┘              │               │ created_at   │
     │ 1                  └───────────────│ updated_at   │
     │                                    └──────┬───────┘
     │                                           │
     │ 1                                         │
┌────┴──────────┐                                │
│  addresses    │                                │
├───────────────┤                                │
│ id (PK)       │                                │
│ user_id (FK)  │                                │
│ street        │                                │
│ city          │                                │
│ state         │                                │
│ zip_code      │                                │
│ is_default    │                                │
└───────────────┘                                │
                                                 │
     ┌───────────────────────────────────────────┘
     │
┌────┴──────────┐       ┌──────────────────┐
│  cart_items   │       │     orders       │
├───────────────┤       ├──────────────────┤
│ id (PK)       │       │ id (PK)          │
│ user_id (FK)──│──→    │ user_id (FK)─────│──→ users.id
│ product_id(FK)│       │ total_amount     │
│ quantity      │       │ status           │  (PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED)
│ added_at      │       │ shipping_address │
└───────────────┘       │ payment_method   │
                        │ created_at       │
                        │ updated_at       │
                        └────────┬─────────┘
                                 │ 1
                                 │
                        ┌────────┴─────────┐
                        │   order_items    │
                        ├──────────────────┤
                        │ id (PK)          │
                        │ order_id (FK)    │
                        │ product_id (FK)  │
                        │ quantity         │
                        │ price_at_purchase│  (snapshot — product price may change later)
                        └──────────────────┘
```

### Key Design Decisions

| Decision | Reasoning |
|----------|-----------|
| **`price_at_purchase` in order_items** | Product price can change. We snapshot it at order time so historical orders remain accurate. |
| **`addresses` as separate table** | A user can have multiple addresses (home, office). One is marked `is_default`. |
| **`cart_items` vs `orders`** | Cart is temporary (can be modified). Order is permanent (immutable after placement). |
| **`status` as ENUM/string** | Order goes through states: PENDING → CONFIRMED → SHIPPED → DELIVERED. Or CANCELLED at any point. |
| **`stock_qty` in products** | Track inventory. Decrease on order placement, increase on cancellation. |

### SQL Schema

```sql
CREATE TABLE users (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(150) UNIQUE NOT NULL,
    password    VARCHAR(255) NOT NULL,
    role        ENUM('CUSTOMER', 'ADMIN') DEFAULT 'CUSTOMER',
    phone       VARCHAR(15),
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE categories (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(200) NOT NULL,
    description TEXT,
    price       DECIMAL(10,2) NOT NULL,
    stock_qty   INT NOT NULL DEFAULT 0,
    image_url   VARCHAR(500),
    category_id BIGINT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (category_id) REFERENCES categories(id)
);

CREATE TABLE addresses (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    street      VARCHAR(255) NOT NULL,
    city        VARCHAR(100) NOT NULL,
    state       VARCHAR(100) NOT NULL,
    zip_code    VARCHAR(20) NOT NULL,
    is_default  BOOLEAN DEFAULT FALSE,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE cart_items (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    product_id  BIGINT NOT NULL,
    quantity    INT NOT NULL DEFAULT 1,
    added_at    TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id),
    FOREIGN KEY (product_id) REFERENCES products(id),
    UNIQUE(user_id, product_id)   -- one entry per product per user
);

CREATE TABLE orders (
    id               BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id          BIGINT NOT NULL,
    total_amount     DECIMAL(10,2) NOT NULL,
    status           ENUM('PENDING','CONFIRMED','SHIPPED','DELIVERED','CANCELLED') DEFAULT 'PENDING',
    shipping_address TEXT NOT NULL,
    payment_method   VARCHAR(50),
    created_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at       TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE order_items (
    id                BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id          BIGINT NOT NULL,
    product_id        BIGINT NOT NULL,
    quantity          INT NOT NULL,
    price_at_purchase DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

### API Endpoints (22 endpoints)

#### Auth (2)
```
POST   /api/auth/register                → Register new user
POST   /api/auth/login                   → Login, returns JWT
```

#### Users (3)
```
GET    /api/users/me                     → Get current user profile
PUT    /api/users/me                     → Update profile
GET    /api/users/me/addresses           → Get user's addresses
POST   /api/users/me/addresses           → Add new address
```

#### Categories (4 — Admin only for CUD)
```
GET    /api/categories                   → List all categories
GET    /api/categories/{id}              → Get category by ID
POST   /api/categories                   → [ADMIN] Create category
PUT    /api/categories/{id}              → [ADMIN] Update category
```

#### Products (5)
```
GET    /api/products                     → List products (paginated, filterable)
GET    /api/products/{id}                → Get product detail
GET    /api/products?category={id}       → Filter by category
POST   /api/products                     → [ADMIN] Create product
PUT    /api/products/{id}                → [ADMIN] Update product
DELETE /api/products/{id}                → [ADMIN] Delete product
```

#### Cart (4)
```
GET    /api/cart                          → Get current user's cart
POST   /api/cart/items                    → Add item to cart
PUT    /api/cart/items/{itemId}           → Update quantity
DELETE /api/cart/items/{itemId}           → Remove item from cart
```

#### Orders (4)
```
POST   /api/orders                        → Place order (from cart)
GET    /api/orders                        → Get user's order history
GET    /api/orders/{id}                   → Get order detail with items
PATCH  /api/orders/{id}/cancel            → Cancel an order
```

### Key Business Rules

```
1. PLACE ORDER:
   - Validate cart is not empty
   - Check stock availability for each item
   - Decrease stock_qty for each product
   - Calculate total (sum of qty × price for each item)
   - Snapshot product prices into order_items.price_at_purchase
   - Clear the cart
   - All of this in ONE @Transactional method (atomic)

2. CANCEL ORDER:
   - Only if status is PENDING or CONFIRMED
   - Restore stock_qty for each item
   - Set status = CANCELLED

3. STOCK CHECK:
   - Before adding to cart: warn if low stock
   - Before placing order: FAIL if insufficient stock
   - Use pessimistic locking or optimistic locking to prevent race conditions
```

### Entity Code (Spring Boot)

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(name = "stock_qty", nullable = false)
    private Integer stockQty;

    @Column(name = "image_url")
    private String imageUrl;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(name = "total_amount", nullable = false)
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @Column(name = "shipping_address", nullable = false)
    private String shippingAddress;

    @Column(name = "payment_method")
    private String paymentMethod;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}

@Entity
@Table(name = "order_items")
public class OrderItem {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id", nullable = false)
    private Product product;

    @Column(nullable = false)
    private Integer quantity;

    @Column(name = "price_at_purchase", nullable = false)
    private BigDecimal priceAtPurchase;
}

public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}
```

### Service — Place Order (Core Business Logic)

```java
@Service
public class OrderService {

    private final OrderRepository orderRepository;
    private final CartItemRepository cartItemRepository;
    private final ProductRepository productRepository;

    @Transactional
    public OrderResponse placeOrder(Long userId, PlaceOrderRequest request) {

        // 1. Get cart items
        List<CartItem> cartItems = cartItemRepository.findByUserId(userId);
        if (cartItems.isEmpty()) {
            throw new BusinessException("Cart is empty");
        }

        // 2. Validate stock & calculate total
        BigDecimal total = BigDecimal.ZERO;
        List<OrderItem> orderItems = new ArrayList<>();

        for (CartItem cartItem : cartItems) {
            Product product = cartItem.getProduct();

            // Stock check
            if (product.getStockQty() < cartItem.getQuantity()) {
                throw new BusinessException(
                    "Insufficient stock for: " + product.getName() +
                    " (available: " + product.getStockQty() + ")");
            }

            // Decrease stock
            product.setStockQty(product.getStockQty() - cartItem.getQuantity());
            productRepository.save(product);

            // Create order item with SNAPSHOT price
            OrderItem orderItem = new OrderItem();
            orderItem.setProduct(product);
            orderItem.setQuantity(cartItem.getQuantity());
            orderItem.setPriceAtPurchase(product.getPrice());  // snapshot!
            orderItems.add(orderItem);

            total = total.add(product.getPrice().multiply(BigDecimal.valueOf(cartItem.getQuantity())));
        }

        // 3. Create order
        Order order = new Order();
        order.setUser(/* current user */);
        order.setTotalAmount(total);
        order.setStatus(OrderStatus.PENDING);
        order.setShippingAddress(request.getShippingAddress());
        order.setPaymentMethod(request.getPaymentMethod());

        orderItems.forEach(item -> item.setOrder(order));
        order.setItems(orderItems);

        Order savedOrder = orderRepository.save(order);

        // 4. Clear cart
        cartItemRepository.deleteByUserId(userId);

        return orderMapper.toResponse(savedOrder);
    }
}
```

---

---

# Scenario 2: Blog / Content Management System

## Requirements

```
A blog platform where:
- Users can register and create a profile
- Users can write, edit, and delete their own posts
- Posts belong to categories and can have multiple tags
- Users can comment on posts
- Users can like/unlike posts
- Posts support pagination and search
```

## Database Design

### ER Diagram (Text)

```
┌──────────┐       ┌──────────────┐       ┌──────────┐
│  users   │       │    posts     │       │ categories│
├──────────┤       ├──────────────┤       ├──────────┤
│ id (PK)  │──┐    │ id (PK)      │   ┌──│ id (PK)  │
│ username │  │    │ title        │   │  │ name     │
│ email    │  │    │ content      │   │  │ slug     │
│ password │  │    │ slug         │   │  └──────────┘
│ bio      │  │    │ published    │   │
│ avatar   │  │    │ author_id(FK)│───┘
│ created_at│ │    │ category_id(FK)──→
└──────────┘  │    │ created_at   │
              │    │ updated_at   │
              │    └──────┬───────┘
              │           │
              │     ┌─────┴──────┐
              │     │            │
              │     │            │
         ┌────┴─────┴──┐   ┌────┴────────┐    ┌──────────┐
         │  comments   │   │ post_tags   │    │   tags   │
         ├─────────────┤   ├─────────────┤    ├──────────┤
         │ id (PK)     │   │ post_id(FK) │───→│ id (PK)  │
         │ post_id(FK) │   │ tag_id(FK)  │    │ name     │
         │ user_id(FK) │   └─────────────┘    │ slug     │
         │ content     │   (Many-to-Many       └──────────┘
         │ created_at  │    join table)
         └─────────────┘
              │
         ┌────┴──────────┐
         │  post_likes   │
         ├───────────────┤
         │ id (PK)       │
         │ post_id (FK)  │
         │ user_id (FK)  │
         │ created_at    │
         └───────────────┘
         UNIQUE(post_id, user_id) — one like per user per post
```

### Key Design Decisions

| Decision | Reasoning |
|----------|-----------|
| **`slug` field on posts** | SEO-friendly URLs: `/posts/how-to-learn-spring-boot` instead of `/posts/42` |
| **`post_tags` join table** | Many-to-Many: one post can have many tags, one tag can be on many posts |
| **`post_likes` with UNIQUE constraint** | Prevents a user from liking a post twice. Toggle: like → unlike → like. |
| **`published` boolean** | Posts can be drafts (false) or published (true). Only published posts are public. |
| **`content` as TEXT/LONGTEXT** | Blog content can be very long. Use TEXT for up to ~65K chars or LONGTEXT for more. |
| **Separate `categories` vs `tags`** | A post belongs to ONE category (hierarchical) but can have MANY tags (flat). |

### SQL Schema

```sql
CREATE TABLE users (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    username    VARCHAR(50) UNIQUE NOT NULL,
    email       VARCHAR(150) UNIQUE NOT NULL,
    password    VARCHAR(255) NOT NULL,
    bio         TEXT,
    avatar_url  VARCHAR(500),
    role        ENUM('USER', 'AUTHOR', 'ADMIN') DEFAULT 'USER',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE categories (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) UNIQUE NOT NULL,
    slug        VARCHAR(100) UNIQUE NOT NULL
);

CREATE TABLE tags (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(50) UNIQUE NOT NULL,
    slug        VARCHAR(50) UNIQUE NOT NULL
);

CREATE TABLE posts (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    title       VARCHAR(300) NOT NULL,
    slug        VARCHAR(300) UNIQUE NOT NULL,
    content     LONGTEXT NOT NULL,
    published   BOOLEAN DEFAULT FALSE,
    author_id   BIGINT NOT NULL,
    category_id BIGINT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (author_id) REFERENCES users(id),
    FOREIGN KEY (category_id) REFERENCES categories(id)
);

-- Many-to-Many join table
CREATE TABLE post_tags (
    post_id     BIGINT NOT NULL,
    tag_id      BIGINT NOT NULL,
    PRIMARY KEY (post_id, tag_id),
    FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
    FOREIGN KEY (tag_id) REFERENCES tags(id)
);

CREATE TABLE comments (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    post_id     BIGINT NOT NULL,
    user_id     BIGINT NOT NULL,
    content     TEXT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE post_likes (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    post_id     BIGINT NOT NULL,
    user_id     BIGINT NOT NULL,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
    FOREIGN KEY (user_id) REFERENCES users(id),
    UNIQUE(post_id, user_id)
);
```

### API Endpoints (20 endpoints)

#### Auth (2)
```
POST   /api/auth/register                → Register
POST   /api/auth/login                   → Login, returns JWT
```

#### Users (3)
```
GET    /api/users/{username}             → Get public profile
PUT    /api/users/me                     → Update own profile
GET    /api/users/me/posts               → Get own posts (including drafts)
```

#### Categories & Tags (4)
```
GET    /api/categories                   → List all categories
GET    /api/tags                         → List all tags
POST   /api/categories                   → [ADMIN] Create category
POST   /api/tags                         → [ADMIN] Create tag
```

#### Posts (7)
```
GET    /api/posts                        → List published posts (paginated)
GET    /api/posts?category={slug}        → Filter by category
GET    /api/posts?tag={slug}             → Filter by tag
GET    /api/posts/search?q={keyword}     → Full-text search
GET    /api/posts/{slug}                 → Get single post by slug
POST   /api/posts                        → Create post (draft or published)
PUT    /api/posts/{slug}                 → Update own post
DELETE /api/posts/{slug}                 → Delete own post
```

#### Comments (3)
```
GET    /api/posts/{slug}/comments        → Get comments for a post
POST   /api/posts/{slug}/comments        → Add comment
DELETE /api/comments/{id}                → Delete own comment
```

#### Likes (1)
```
POST   /api/posts/{slug}/like            → Toggle like/unlike
```

### Entity — Many-to-Many (Posts ↔ Tags)

```java
@Entity
@Table(name = "posts")
public class Post {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(nullable = false, unique = true)
    private String slug;

    @Column(nullable = false, columnDefinition = "LONGTEXT")
    private String content;

    private boolean published;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id", nullable = false)
    private User author;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", nullable = false)
    private Category category;

    // Many-to-Many with tags
    @ManyToMany
    @JoinTable(
        name = "post_tags",
        joinColumns = @JoinColumn(name = "post_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    private Set<Tag> tags = new HashSet<>();

    @OneToMany(mappedBy = "post", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Comment> comments = new ArrayList<>();

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

### Like Toggle Service (Interesting Logic)

```java
@Service
public class LikeService {

    private final PostLikeRepository likeRepository;

    @Transactional
    public LikeResponse toggleLike(Long userId, Long postId) {
        Optional<PostLike> existingLike = likeRepository.findByUserIdAndPostId(userId, postId);

        if (existingLike.isPresent()) {
            // Already liked → UNLIKE
            likeRepository.delete(existingLike.get());
            long count = likeRepository.countByPostId(postId);
            return new LikeResponse(false, count);  // liked=false
        } else {
            // Not liked → LIKE
            PostLike like = new PostLike();
            like.setUserId(userId);
            like.setPostId(postId);
            likeRepository.save(like);
            long count = likeRepository.countByPostId(postId);
            return new LikeResponse(true, count);  // liked=true
        }
    }
}
```

---

---

# Scenario 3: Employee Management System (HR Portal)

## Requirements

```
An HR portal where:
- Admins can manage departments
- Admins can add, update, and deactivate employees
- Employees belong to a department and report to a manager
- Track employee salary history
- Leave management (apply, approve, reject)
- Generate reports (employees per department, salary statistics)
```

## Database Design

### ER Diagram (Text)

```
┌──────────────┐           ┌──────────────┐
│ departments  │           │  employees   │
├──────────────┤           ├──────────────┤
│ id (PK)      │◄──────────│ id (PK)      │
│ name         │  dept_id  │ first_name   │
│ code         │ (FK)      │ last_name    │
│ description  │           │ email        │
│ created_at   │   ┌──────│ manager_id(FK)│──→ employees.id (SELF-REFERENCE)
└──────────────┘   │       │ department_id(FK)
                   │       │ designation  │
                   │       │ salary       │
                   │       │ hire_date    │
                   │       │ status       │ (ACTIVE, INACTIVE, ON_LEAVE)
                   │       │ created_at   │
                   │       └──────┬───────┘
                   │              │
                   │    ┌─────────┴────────────┐
                   │    │                      │
            ┌──────┴────┴──┐          ┌────────┴───────┐
            │salary_history│          │    leaves      │
            ├──────────────┤          ├────────────────┤
            │ id (PK)      │          │ id (PK)        │
            │ employee_id  │          │ employee_id(FK)│
            │ old_salary   │          │ leave_type     │ (ANNUAL, SICK, CASUAL)
            │ new_salary   │          │ start_date     │
            │ change_reason│          │ end_date       │
            │ changed_at   │          │ reason         │
            │ changed_by   │          │ status         │ (PENDING, APPROVED, REJECTED)
            └──────────────┘          │ approved_by(FK)│──→ employees.id
                                      │ created_at     │
                                      └────────────────┘
```

### Key Design Decisions

| Decision | Reasoning |
|----------|-----------|
| **`manager_id` self-reference** | An employee's manager IS another employee. This creates a tree hierarchy. CEO has `manager_id = NULL`. |
| **`salary_history` table** | Never overwrite salary — keep an audit trail. Current salary is in `employees`, history tracks changes. |
| **`status` on employee** | Soft delete — don't actually delete employees. Mark as INACTIVE. Historical data is preserved. |
| **`leave_type` as ENUM** | Fixed set of leave types. Could also be a separate `leave_types` table for flexibility. |
| **`approved_by` on leaves** | Track WHO approved/rejected the leave (the manager). |

### API Endpoints (18 endpoints)

#### Departments (4)
```
GET    /api/departments                  → List all departments
GET    /api/departments/{id}             → Get department with employee count
POST   /api/departments                  → [ADMIN] Create department
PUT    /api/departments/{id}             → [ADMIN] Update department
```

#### Employees (7)
```
GET    /api/employees                    → List employees (paginated, filterable by dept/status)
GET    /api/employees/{id}               → Get employee detail (with department, manager info)
POST   /api/employees                    → [ADMIN] Add new employee
PUT    /api/employees/{id}               → [ADMIN] Update employee
PATCH  /api/employees/{id}/deactivate    → [ADMIN] Soft-delete (set INACTIVE)
GET    /api/employees/{id}/subordinates  → Get direct reports (who reports to this employee)
PUT    /api/employees/{id}/salary        → [ADMIN] Update salary (creates salary_history record)
```

#### Salary History (1)
```
GET    /api/employees/{id}/salary-history → Get salary change history
```

#### Leaves (5)
```
POST   /api/leaves                       → Apply for leave
GET    /api/leaves/me                    → Get my leave requests
GET    /api/leaves/pending               → [MANAGER] Get pending leaves for subordinates
PATCH  /api/leaves/{id}/approve          → [MANAGER] Approve leave
PATCH  /api/leaves/{id}/reject           → [MANAGER] Reject leave
```

#### Reports (1)
```
GET    /api/reports/department-summary    → Employee count + avg salary per department
```

### Salary Update Service (Audit Trail)

```java
@Service
public class EmployeeService {

    @Transactional
    public EmployeeResponse updateSalary(Long employeeId, SalaryUpdateRequest request, Long changedByUserId) {

        Employee employee = employeeRepository.findById(employeeId)
                .orElseThrow(() -> new ResourceNotFoundException("Employee", "id", employeeId));

        // 1. Save history BEFORE changing
        SalaryHistory history = new SalaryHistory();
        history.setEmployee(employee);
        history.setOldSalary(employee.getSalary());
        history.setNewSalary(request.getNewSalary());
        history.setChangeReason(request.getReason());
        history.setChangedBy(changedByUserId);
        history.setChangedAt(LocalDateTime.now());
        salaryHistoryRepository.save(history);

        // 2. Update current salary
        employee.setSalary(request.getNewSalary());
        employeeRepository.save(employee);

        return employeeMapper.toResponse(employee);
    }
}
```

### Self-Referencing Entity (Manager Hierarchy)

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String firstName;
    private String lastName;
    private String email;
    private String designation;
    private BigDecimal salary;
    private LocalDate hireDate;

    @Enumerated(EnumType.STRING)
    private EmployeeStatus status;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;

    // SELF-REFERENCE: an employee's manager is also an employee
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "manager_id")
    private Employee manager;

    // Employees who report to THIS employee
    @OneToMany(mappedBy = "manager")
    private List<Employee> subordinates = new ArrayList<>();
}
```

---

---

# Scenario 4: Food Delivery App

## Requirements

```
A food delivery app where:
- Customers can browse restaurants and menus
- Customers can place orders from a restaurant
- Restaurant owners can manage their menu
- Delivery agents can accept and complete deliveries
- Real-time order status tracking
- Ratings and reviews for restaurants
```

## Database Design

### ER Diagram (Text)

```
┌──────────┐         ┌───────────────┐         ┌──────────────┐
│  users   │         │  restaurants  │         │  menu_items  │
├──────────┤         ├───────────────┤         ├──────────────┤
│ id (PK)  │    ┌───│ id (PK)       │◄────────│ id (PK)      │
│ name     │    │    │ name          │  rest_id│ restaurant_id│
│ email    │    │    │ description   │   (FK)  │ name         │
│ password │    │    │ address       │         │ description  │
│ phone    │    │    │ cuisine_type  │         │ price        │
│ role     │    │    │ owner_id (FK) │──→users │ category     │ (Starters, Mains, Desserts, Drinks)
│ created_at│   │    │ rating        │         │ image_url    │
└──────┬───┘    │    │ is_open       │         │ is_available │
       │        │    │ latitude      │         │ created_at   │
       │        │    │ longitude     │         └──────────────┘
       │        │    │ created_at    │
       │        │    └───────────────┘
       │        │
       │        │
  ┌────┴────────┴──────────┐        ┌──────────────────┐
  │      orders            │        │  delivery_agents │
  ├────────────────────────┤        ├──────────────────┤
  │ id (PK)                │   ┌───│ id (PK)          │
  │ customer_id (FK)──→users│   │   │ user_id (FK)     │
  │ restaurant_id (FK)     │   │   │ is_available      │
  │ agent_id (FK)──────────│───┘   │ current_lat       │
  │ total_amount           │       │ current_lng       │
  │ delivery_fee           │       │ vehicle_type      │
  │ status                 │       └──────────────────┘
  │ delivery_address       │        (PLACED → ACCEPTED → PREPARING →
  │ special_instructions   │         READY → PICKED_UP → DELIVERED)
  │ estimated_delivery     │        or CANCELLED at any point
  │ created_at             │
  │ updated_at             │
  └───────────┬────────────┘
              │ 1
              │
     ┌────────┴─────────┐          ┌──────────────┐
     │   order_items    │          │   reviews    │
     ├──────────────────┤          ├──────────────┤
     │ id (PK)          │          │ id (PK)      │
     │ order_id (FK)    │          │ user_id (FK) │
     │ menu_item_id(FK) │          │ restaurant_id│
     │ quantity         │          │ order_id(FK) │
     │ price            │          │ rating (1-5) │
     │ notes            │          │ comment      │
     └──────────────────┘          │ created_at   │
                                   └──────────────┘
                                   UNIQUE(user_id, order_id)
```

### Key Design Decisions

| Decision | Reasoning |
|----------|-----------|
| **Separate `delivery_agents`** | Not all users are agents. Agent-specific fields (location, availability, vehicle) don't belong in the users table. |
| **`rating` on restaurant** | Denormalized — pre-calculated average. Updated on every new review. Avoids calculating AVG on every read. |
| **`is_open` on restaurant** | Restaurant can be temporarily closed. Menu items still exist but orders can't be placed. |
| **`latitude/longitude`** | For distance-based restaurant search ("restaurants near me") and delivery tracking. |
| **`estimated_delivery` on order** | Set when agent picks up. Updated in real-time. |
| **Review tied to `order_id`** | Can only review after completing an order. UNIQUE constraint prevents duplicate reviews per order. |
| **Order status as state machine** | PLACED → ACCEPTED → PREPARING → READY → PICKED_UP → DELIVERED. Clear, trackable transitions. |

### API Endpoints (24 endpoints)

#### Auth (2)
```
POST   /api/auth/register                → Register (customer, owner, or agent)
POST   /api/auth/login                   → Login
```

#### Restaurants (5)
```
GET    /api/restaurants                  → List restaurants (paginated, filter by cuisine, search by name)
GET    /api/restaurants?lat=x&lng=y&radius=5  → Nearby restaurants
GET    /api/restaurants/{id}             → Get restaurant detail with menu
POST   /api/restaurants                  → [OWNER] Register restaurant
PUT    /api/restaurants/{id}             → [OWNER] Update restaurant info
PATCH  /api/restaurants/{id}/toggle      → [OWNER] Open/close restaurant
```

#### Menu Items (4)
```
GET    /api/restaurants/{id}/menu        → Get full menu
POST   /api/restaurants/{id}/menu        → [OWNER] Add menu item
PUT    /api/menu-items/{itemId}          → [OWNER] Update item
DELETE /api/menu-items/{itemId}          → [OWNER] Remove item
```

#### Orders (6)
```
POST   /api/orders                       → Place order
GET    /api/orders                       → My order history (customer)
GET    /api/orders/{id}                  → Order detail with tracking
GET    /api/orders/{id}/track            → Real-time delivery status
PATCH  /api/orders/{id}/cancel           → Cancel order
GET    /api/restaurants/{id}/orders      → [OWNER] Restaurant's incoming orders
PATCH  /api/orders/{id}/status           → [OWNER/AGENT] Update order status
```

#### Delivery (3)
```
GET    /api/deliveries/available         → [AGENT] Available orders to pick up
PATCH  /api/deliveries/{orderId}/accept  → [AGENT] Accept delivery
PATCH  /api/deliveries/{orderId}/complete→ [AGENT] Mark as delivered
```

#### Reviews (3)
```
POST   /api/orders/{orderId}/review      → Submit review (after delivery)
GET    /api/restaurants/{id}/reviews     → Get restaurant reviews (paginated)
DELETE /api/reviews/{id}                 → Delete own review
```

#### Reports (1)
```
GET    /api/restaurants/{id}/stats       → [OWNER] Order count, revenue, avg rating
```

### Order Status State Machine

```
    PLACED ──→ ACCEPTED ──→ PREPARING ──→ READY ──→ PICKED_UP ──→ DELIVERED
      │            │            │           │           │
      └────────────┴────────────┴───────────┘           │
                        │                               │
                    CANCELLED                      (can't cancel
                                                   after pickup)
```

```java
@Service
public class OrderStatusService {

    private static final Map<OrderStatus, Set<OrderStatus>> VALID_TRANSITIONS = Map.of(
        OrderStatus.PLACED,     Set.of(OrderStatus.ACCEPTED, OrderStatus.CANCELLED),
        OrderStatus.ACCEPTED,   Set.of(OrderStatus.PREPARING, OrderStatus.CANCELLED),
        OrderStatus.PREPARING,  Set.of(OrderStatus.READY, OrderStatus.CANCELLED),
        OrderStatus.READY,      Set.of(OrderStatus.PICKED_UP),
        OrderStatus.PICKED_UP,  Set.of(OrderStatus.DELIVERED)
    );

    @Transactional
    public void updateStatus(Long orderId, OrderStatus newStatus) {
        Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new ResourceNotFoundException("Order", "id", orderId));

        Set<OrderStatus> allowed = VALID_TRANSITIONS.get(order.getStatus());
        if (allowed == null || !allowed.contains(newStatus)) {
            throw new BusinessException(
                "Cannot transition from " + order.getStatus() + " to " + newStatus);
        }

        order.setStatus(newStatus);
        orderRepository.save(order);
    }
}
```

---

---

# Scenario 5: Banking / Wallet System

## Requirements

```
A digital wallet / banking system where:
- Users can create an account with a wallet
- Users can deposit and withdraw money
- Users can transfer money to other users
- Complete transaction history with audit trail
- Balance should NEVER go negative
- All money operations must be atomic (no partial transfers)
```

## Database Design

### ER Diagram (Text)

```
┌──────────────┐           ┌──────────────┐
│   accounts   │           │   wallets    │
├──────────────┤           ├──────────────┤
│ id (PK)      │◄──1:1────│ id (PK)      │
│ name         │           │ account_id(FK)│ UNIQUE
│ email        │           │ balance      │  DECIMAL(15,2) — NOT negative
│ password     │           │ currency     │  (INR, USD, EUR)
│ phone        │           │ created_at   │
│ is_active    │           │ updated_at   │
│ created_at   │           └──────┬───────┘
└──────────────┘                  │
                                  │
                         ┌────────┴─────────┐
                         │  transactions   │
                         ├──────────────────┤
                         │ id (PK)          │
                         │ reference_id     │  UUID — unique per transaction (idempotency)
                         │ type             │  (DEPOSIT, WITHDRAWAL, TRANSFER_IN, TRANSFER_OUT)
                         │ amount           │  DECIMAL(15,2)
                         │ balance_after    │  Snapshot of balance AFTER this transaction
                         │ wallet_id (FK)   │
                         │ counterparty_wallet_id │  (for transfers — who sent/received)
                         │ description      │
                         │ status           │  (SUCCESS, FAILED, PENDING)
                         │ created_at       │
                         └──────────────────┘
```

### Key Design Decisions

| Decision | Reasoning |
|----------|-----------|
| **Separate `wallets` table** | Wallet has its own lifecycle. An account could potentially have multiple wallets (multi-currency). |
| **`balance` with CHECK constraint** | `CHECK (balance >= 0)` — database enforces that balance NEVER goes negative, even if code has a bug. |
| **`balance_after` in transactions** | Snapshot of balance after each transaction. Enables recreating balance history without reprocessing all transactions. Also serves as audit proof. |
| **`reference_id` (UUID)** | Idempotency key. If a transfer request is sent twice (network retry), the second one is detected as duplicate and ignored. |
| **`type` with TRANSFER_IN / TRANSFER_OUT** | A transfer between two users creates TWO transaction records — one TRANSFER_OUT for the sender, one TRANSFER_IN for the receiver. Each wallet sees its own side. |
| **`DECIMAL(15,2)`** | NEVER use `float`/`double` for money! Floating point has rounding errors. `DECIMAL` is exact. |
| **Pessimistic locking** | When updating balance, use `SELECT ... FOR UPDATE` to prevent race conditions (two withdrawals at the same time). |

### SQL Schema

```sql
CREATE TABLE accounts (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(150) UNIQUE NOT NULL,
    password    VARCHAR(255) NOT NULL,
    phone       VARCHAR(15),
    is_active   BOOLEAN DEFAULT TRUE,
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE wallets (
    id          BIGINT AUTO_INCREMENT PRIMARY KEY,
    account_id  BIGINT UNIQUE NOT NULL,
    balance     DECIMAL(15,2) NOT NULL DEFAULT 0.00,
    currency    VARCHAR(3) DEFAULT 'INR',
    created_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at  TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES accounts(id),
    CHECK (balance >= 0)
);

CREATE TABLE transactions (
    id                      BIGINT AUTO_INCREMENT PRIMARY KEY,
    reference_id            VARCHAR(36) UNIQUE NOT NULL,    -- UUID for idempotency
    type                    ENUM('DEPOSIT','WITHDRAWAL','TRANSFER_IN','TRANSFER_OUT') NOT NULL,
    amount                  DECIMAL(15,2) NOT NULL,
    balance_after           DECIMAL(15,2) NOT NULL,
    wallet_id               BIGINT NOT NULL,
    counterparty_wallet_id  BIGINT,                         -- NULL for deposit/withdrawal
    description             VARCHAR(255),
    status                  ENUM('SUCCESS','FAILED','PENDING') DEFAULT 'SUCCESS',
    created_at              TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (wallet_id) REFERENCES wallets(id),
    FOREIGN KEY (counterparty_wallet_id) REFERENCES wallets(id),
    CHECK (amount > 0)
);

-- INDEX for fast lookups
CREATE INDEX idx_transactions_wallet ON transactions(wallet_id, created_at DESC);
CREATE INDEX idx_transactions_ref ON transactions(reference_id);
```

### API Endpoints (10 endpoints)

#### Auth (2)
```
POST   /api/auth/register                → Create account + wallet
POST   /api/auth/login                   → Login
```

#### Wallet (3)
```
GET    /api/wallet                       → Get my wallet (balance, currency)
POST   /api/wallet/deposit               → Deposit money
POST   /api/wallet/withdraw              → Withdraw money
```

#### Transfers (2)
```
POST   /api/transfers                    → Transfer to another user
GET    /api/transfers/{referenceId}       → Get transfer status (idempotency check)
```

#### Transactions (2)
```
GET    /api/transactions                 → My transaction history (paginated)
GET    /api/transactions/{id}            → Transaction detail
```

#### Account (1)
```
GET    /api/account/me                   → My account info
```

### Transfer Service — The Most Critical Code

```java
@Service
public class TransferService {

    private final WalletRepository walletRepository;
    private final TransactionRepository transactionRepository;

    @Transactional(isolation = Isolation.SERIALIZABLE)  // Highest isolation for money
    public TransferResponse transfer(Long senderAccountId, TransferRequest request) {

        // 0. Idempotency check — has this transfer already been processed?
        if (transactionRepository.existsByReferenceId(request.getReferenceId())) {
            Transaction existing = transactionRepository.findByReferenceId(request.getReferenceId());
            return new TransferResponse(existing, "Transfer already processed");
        }

        // 1. Load wallets WITH PESSIMISTIC LOCK (prevents race conditions)
        Wallet senderWallet = walletRepository.findByAccountIdWithLock(senderAccountId)
                .orElseThrow(() -> new ResourceNotFoundException("Wallet", "accountId", senderAccountId));

        Wallet receiverWallet = walletRepository.findByAccountIdWithLock(request.getReceiverAccountId())
                .orElseThrow(() -> new ResourceNotFoundException("Wallet", "accountId", request.getReceiverAccountId()));

        // 2. Validate
        BigDecimal amount = request.getAmount();

        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new BusinessException("Transfer amount must be positive");
        }

        if (senderWallet.getId().equals(receiverWallet.getId())) {
            throw new BusinessException("Cannot transfer to yourself");
        }

        if (senderWallet.getBalance().compareTo(amount) < 0) {
            throw new BusinessException("Insufficient balance. Available: " + senderWallet.getBalance());
        }

        // 3. Update balances
        senderWallet.setBalance(senderWallet.getBalance().subtract(amount));
        receiverWallet.setBalance(receiverWallet.getBalance().add(amount));

        walletRepository.save(senderWallet);
        walletRepository.save(receiverWallet);

        // 4. Create transaction records (both sides)
        Transaction senderTxn = new Transaction();
        senderTxn.setReferenceId(request.getReferenceId());
        senderTxn.setType(TransactionType.TRANSFER_OUT);
        senderTxn.setAmount(amount);
        senderTxn.setBalanceAfter(senderWallet.getBalance());
        senderTxn.setWallet(senderWallet);
        senderTxn.setCounterpartyWallet(receiverWallet);
        senderTxn.setDescription("Transfer to account #" + request.getReceiverAccountId());
        senderTxn.setStatus(TransactionStatus.SUCCESS);

        Transaction receiverTxn = new Transaction();
        receiverTxn.setReferenceId(UUID.randomUUID().toString());  // Different ref for receiver's record
        receiverTxn.setType(TransactionType.TRANSFER_IN);
        receiverTxn.setAmount(amount);
        receiverTxn.setBalanceAfter(receiverWallet.getBalance());
        receiverTxn.setWallet(receiverWallet);
        receiverTxn.setCounterpartyWallet(senderWallet);
        receiverTxn.setDescription("Transfer from account #" + senderAccountId);
        receiverTxn.setStatus(TransactionStatus.SUCCESS);

        transactionRepository.save(senderTxn);
        transactionRepository.save(receiverTxn);

        return new TransferResponse(senderTxn, "Transfer successful");
    }
}
```

### Pessimistic Locking in Repository

```java
@Repository
public interface WalletRepository extends JpaRepository<Wallet, Long> {

    // PESSIMISTIC_WRITE = SELECT ... FOR UPDATE
    // This LOCKS the row until the transaction completes
    // Other transactions trying to read this row will WAIT
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT w FROM Wallet w WHERE w.account.id = :accountId")
    Optional<Wallet> findByAccountIdWithLock(@Param("accountId") Long accountId);

    Optional<Wallet> findByAccountId(Long accountId);
}
```

### Why Pessimistic Locking for Money?

```
WITHOUT locking (race condition):
─────────────────────────────────
  Balance = $100

  Thread A: reads balance = $100, withdraws $80 → sets balance = $20
  Thread B: reads balance = $100, withdraws $80 → sets balance = $20

  Both succeed! But total withdrawn = $160 from a $100 balance! 💥

WITH pessimistic locking:
─────────────────────────
  Balance = $100

  Thread A: LOCKS the row, reads balance = $100, withdraws $80 → balance = $20, UNLOCKS
  Thread B: WAITS for lock... gets lock, reads balance = $20, withdraws $80 → FAILS (insufficient) ✅
```

---

---

# Summary — DB Design Principles Across All Scenarios

## Universal Rules

| Rule | Explanation | Example |
|------|-------------|---------|
| **Never use `float`/`double` for money** | Floating point has rounding errors | Use `DECIMAL(15,2)` or `BigDecimal` in Java |
| **Snapshot prices at transaction time** | Prices change; historical records must be accurate | `order_items.price_at_purchase` |
| **Soft delete > Hard delete** | Preserve data for audits, analytics, compliance | `is_active = false` instead of `DELETE` |
| **Audit trails for sensitive changes** | Track who changed what and when | `salary_history`, `transactions.balance_after` |
| **Use UUIDs for external IDs** | Auto-increment IDs leak information (how many orders you have) | `reference_id` in transactions |
| **Denormalize for read performance** | Pre-calculate frequently read values | `restaurant.rating` (average of reviews) |
| **Use ENUM or status columns** | State machines for entities that go through stages | `order.status`, `leave.status` |
| **Index foreign keys and search columns** | Without indexes, JOINs and WHERE clauses are slow | `INDEX on wallet_id`, `INDEX on email` |
| **Use constraints at DB level** | Don't trust application code alone | `CHECK (balance >= 0)`, `UNIQUE(email)` |
| **Separate concerns into tables** | Don't cram everything into one table | `wallets` separate from `accounts` |

## Relationship Patterns Cheat Sheet

| Pattern | Example | JPA Annotation |
|---------|---------|---------------|
| **One-to-One** | Account ↔ Wallet | `@OneToOne` |
| **One-to-Many** | Department → Employees | `@OneToMany` / `@ManyToOne` |
| **Many-to-Many** | Posts ↔ Tags | `@ManyToMany` + `@JoinTable` |
| **Self-Reference** | Employee → Manager (also Employee) | `@ManyToOne` on same entity |

## Endpoint Count by Project Size

| Project Size | Typical Endpoints | Example |
|-------------|-------------------|---------|
| Small (CRUD) | 8-12 | Blog, Todo app |
| Medium | 15-25 | HR Portal, CMS |
| Large | 25-40+ | E-Commerce, Food Delivery |
| Enterprise | 50+ | Banking, ERP |

---

### Interview Tip 🗣️

> "When designing a backend, I start with **entities and their relationships** (ER diagram), then derive the **API endpoints** from the use cases. I always snapshot prices/amounts at transaction time, use soft deletes for audit compliance, add DB-level constraints as a safety net, and use pessimistic locking for any financial operations. The number of endpoints depends on the domain — a CRUD app needs ~10, an e-commerce platform needs ~25."

---

[⬅️ Back to Curriculum](./README.md) | [⬅️ Previous: Microservices vs Monolith](./10-microservices-vs-monolith.md)
