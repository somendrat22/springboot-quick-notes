# Module 7: Spring Boot Annotations Cheat Sheet

---

## 7.1 Why Annotations?

### Layman Example: Sticky Notes on a Whiteboard 📌

Imagine a big office whiteboard with employee names. You put **sticky notes** on each name:

```
┌──────────────────────────────────────────┐
│              OFFICE WHITEBOARD            │
│                                          │
│   John  [MANAGER] [FLOOR-3]             │
│   Alice [DEVELOPER] [FLOOR-2] [ON-CALL] │
│   Bob   [INTERN] [FLOOR-1]             │
│                                          │
└──────────────────────────────────────────┘
```

- The sticky notes don't change WHO the person is — they add **metadata**.
- The HR system reads the sticky notes to decide: who gets which access badge, which floor, which responsibilities.

**Annotations in Java work the same way:**
- They're "sticky notes" on your classes, methods, or fields.
- Spring reads them at startup to decide: what to create, how to wire, where to route requests.

---

## 7.2 Core Spring / Spring Boot Annotations

### Application Setup

| Annotation | Where | What It Does |
|-----------|-------|-------------|
| `@SpringBootApplication` | Main class | Combines `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`. The entry point of your app. |
| `@Configuration` | Class | Marks a class as a source of bean definitions (like an XML config, but in Java). |
| `@ComponentScan` | Class | Tells Spring which packages to scan for `@Component`, `@Service`, etc. |
| `@EnableAutoConfiguration` | Class | Spring Boot magic — auto-configures beans based on classpath dependencies. |

```java
@SpringBootApplication  // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

### Bean Registration

| Annotation | Where | What It Does |
|-----------|-------|-------------|
| `@Component` | Class | Generic — "Register this class as a Spring bean." |
| `@Service` | Class | Same as `@Component` but signals "I contain business logic." |
| `@Repository` | Class | Same as `@Component` but signals "I access the database." Also adds automatic exception translation. |
| `@Controller` | Class | Same as `@Component` but signals "I handle web requests (returns views)." |
| `@RestController` | Class | `@Controller` + `@ResponseBody` — returns JSON/XML instead of views. |
| `@Bean` | Method | Registers the method's return value as a bean. Used inside `@Configuration` classes. |

```java
// @Component family — Spring auto-detects via component scanning
@Service
public class OrderService { }

// @Bean — manual registration inside a config class
@Configuration
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();  // This object is now a Spring-managed bean
    }

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
                .registerModule(new JavaTimeModule())
                .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }
}
```

### When to Use `@Component` vs `@Bean`

| Scenario | Use |
|----------|-----|
| **Your own class** that you wrote | `@Component` (or `@Service`, `@Repository`, `@Controller`) |
| **Third-party class** you can't annotate (e.g., `RestTemplate`, `ObjectMapper`) | `@Bean` inside a `@Configuration` class |
| Need **custom construction logic** | `@Bean` — you control how the object is created |

---

## 7.3 Dependency Injection Annotations

| Annotation | Where | What It Does |
|-----------|-------|-------------|
| `@Autowired` | Constructor, Setter, Field | "Inject a bean here." Optional on constructors if there's only one. |
| `@Qualifier("name")` | Parameter, Field | "Inject THIS specific bean by name" (when multiple beans of same type exist). |
| `@Primary` | Class or `@Bean` method | "If there are multiple beans of this type, prefer me by default." |
| `@Value("${prop}")` | Field, Parameter | Inject a value from `application.properties`. |
| `@Lazy` | Class, Field | Don't create this bean at startup — create it on first use. |

```java
// @Value — inject config values
@Service
public class EmailService {

    @Value("${app.email.sender}")          // from application.properties
    private String senderEmail;

    @Value("${app.email.max-retries:3}")   // default value = 3 if property missing
    private int maxRetries;

    @Value("${app.feature.notifications-enabled:true}")
    private boolean notificationsEnabled;
}
```

```properties
# application.properties
app.email.sender=noreply@example.com
app.email.max-retries=5
app.feature.notifications-enabled=true
```

---

## 7.4 Web / REST Annotations

### Class-Level

| Annotation | What It Does |
|-----------|-------------|
| `@RestController` | This class handles HTTP requests and returns JSON |
| `@RequestMapping("/api/orders")` | Base URL path for all endpoints in this controller |
| `@CrossOrigin` | Allow cross-origin requests (CORS) |

### Method-Level — HTTP Verbs

| Annotation | HTTP Method | Example |
|-----------|-------------|---------|
| `@GetMapping("/path")` | GET | `@GetMapping("/{id}")` |
| `@PostMapping("/path")` | POST | `@PostMapping` |
| `@PutMapping("/path")` | PUT | `@PutMapping("/{id}")` |
| `@PatchMapping("/path")` | PATCH | `@PatchMapping("/{id}")` |
| `@DeleteMapping("/path")` | DELETE | `@DeleteMapping("/{id}")` |

### Parameter-Level

| Annotation | What It Does | Example |
|-----------|-------------|---------|
| `@PathVariable` | Extracts from URL path | `/orders/{id}` → `@PathVariable Long id` |
| `@RequestParam` | Extracts from query string | `?name=John` → `@RequestParam String name` |
| `@RequestBody` | Deserializes JSON body → Java object | `@RequestBody OrderRequest req` |
| `@RequestHeader` | Extracts an HTTP header value | `@RequestHeader("Authorization") String token` |
| `@CookieValue` | Extracts a cookie value | `@CookieValue("session") String sessionId` |

```java
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    // All annotations in action:
    @GetMapping("/search")
    public ResponseEntity<List<Employee>> search(
            @RequestParam String department,                        // ?department=Engineering
            @RequestParam(defaultValue = "0") int page,            // ?page=0 (optional, default 0)
            @RequestParam(required = false) String sortBy,         // ?sortBy=salary (optional)
            @RequestHeader("X-Request-Id") String requestId) {     // Custom header

        // ...
    }

    @PostMapping
    public ResponseEntity<Employee> create(
            @Valid @RequestBody EmployeeCreateRequest request) {   // JSON body → Java object
        // ...
    }

    @GetMapping("/{id}")
    public ResponseEntity<Employee> getById(
            @PathVariable Long id) {                               // /employees/42 → id = 42
        // ...
    }
}
```

---

## 7.5 Validation Annotations

| Annotation | What It Validates |
|-----------|-------------------|
| `@Valid` | Triggers validation on a `@RequestBody` |
| `@NotNull` | Not null |
| `@NotBlank` | Not null + not empty + not just whitespace (for strings) |
| `@NotEmpty` | Not null + not empty (for strings, collections) |
| `@Size(min, max)` | String length or collection size |
| `@Min(value)` | Number >= value |
| `@Max(value)` | Number <= value |
| `@Email` | Valid email format |
| `@Pattern(regexp)` | Matches a regex |
| `@Positive` / `@PositiveOrZero` | Number > 0 / >= 0 |
| `@Past` / `@Future` | Date is in the past / future |

---

## 7.6 JPA / Database Annotations

### Entity Mapping

| Annotation | Where | What It Does |
|-----------|-------|-------------|
| `@Entity` | Class | "This class maps to a database table" |
| `@Table(name = "...")` | Class | Specify the table name (defaults to class name) |
| `@Id` | Field | "This is the primary key" |
| `@GeneratedValue` | Field | Auto-generate the ID (IDENTITY, SEQUENCE, UUID, etc.) |
| `@Column` | Field | Customize column name, nullable, unique, length |
| `@Transient` | Field | "Don't persist this field to the database" |
| `@Enumerated` | Field | How to store enums (STRING or ORDINAL) |
| `@CreationTimestamp` | Field | Auto-set on entity creation |
| `@UpdateTimestamp` | Field | Auto-set on entity update |

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    private String fullName;

    @Column(unique = true, nullable = false)
    private String email;

    @Enumerated(EnumType.STRING)       // Store as "ACTIVE" not 0
    private Status status;

    @Transient                          // Computed field — not saved to DB
    private String displayName;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}
```

### Relationship Mapping

| Annotation | What It Does | Example |
|-----------|-------------|---------|
| `@OneToOne` | One-to-one relationship | Employee ↔ Address |
| `@OneToMany` | One-to-many | Department → List\<Employee\> |
| `@ManyToOne` | Many-to-one | Employee → Department |
| `@ManyToMany` | Many-to-many | Student ↔ Course |
| `@JoinColumn` | Specifies the FK column | `@JoinColumn(name = "dept_id")` |
| `@JoinTable` | Specifies the join table (for M:N) | `@JoinTable(name = "student_course")` |

---

## 7.7 Transaction & Caching Annotations

| Annotation | What It Does |
|-----------|-------------|
| `@Transactional` | Wraps the method in a DB transaction (all-or-nothing). Place on Service layer. |
| `@Transactional(readOnly = true)` | Optimization hint for read-only operations. |
| `@Cacheable("cacheName")` | Cache the result — next call returns cached value. |
| `@CacheEvict("cacheName")` | Remove entry from cache (on update/delete). |
| `@CachePut("cacheName")` | Update the cache with new value. |

```java
@Service
public class ProductService {

    @Cacheable("products")              // First call → DB query. Next calls → cached.
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    @CacheEvict(value = "products", key = "#id")  // Remove from cache on update
    @Transactional
    public Product updateProduct(Long id, Product product) {
        // ...
    }

    @Transactional(readOnly = true)     // Hint: no writes, can optimize
    public List<Product> getAllProducts() {
        return productRepository.findAll();
    }
}
```

---

## 7.8 Profile & Conditional Annotations

| Annotation | What It Does |
|-----------|-------------|
| `@Profile("dev")` | This bean is ONLY created when the "dev" profile is active |
| `@ConditionalOnProperty` | Create bean only if a specific property is set |
| `@ConditionalOnClass` | Create bean only if a specific class is on the classpath |

```java
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        // H2 in-memory database for development
        return new EmbeddedDatabaseBuilder()
                .setType(EmbeddedDatabaseType.H2)
                .build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        // Real MySQL for production
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://prod-server:3306/mydb");
        return ds;
    }
}
```

```properties
# Activate a profile
# application.properties
spring.profiles.active=dev

# Or via command line:
# java -jar app.jar --spring.profiles.active=prod
```

---

## 7.9 Scheduling & Async Annotations

| Annotation | What It Does |
|-----------|-------------|
| `@Scheduled(fixedRate = 5000)` | Run this method every 5 seconds |
| `@Scheduled(cron = "0 0 9 * * MON")` | Cron expression — every Monday at 9 AM |
| `@Async` | Run this method in a separate thread (non-blocking) |
| `@EnableScheduling` | Enable `@Scheduled` support (put on main class or config) |
| `@EnableAsync` | Enable `@Async` support |

```java
@Service
@EnableScheduling
public class ReportService {

    @Scheduled(cron = "0 0 6 * * *")     // Every day at 6 AM
    public void generateDailyReport() {
        // ...
    }

    @Async                                // Runs in background thread
    public CompletableFuture<Report> generateLargeReport() {
        // Long-running task...
        return CompletableFuture.completedFuture(report);
    }
}
```

---

## 7.10 Exception Handling Annotations

| Annotation | What It Does |
|-----------|-------------|
| `@ControllerAdvice` | Global exception handler for all controllers |
| `@RestControllerAdvice` | `@ControllerAdvice` + `@ResponseBody` (returns JSON) |
| `@ExceptionHandler(XException.class)` | Handle a specific exception type |
| `@ResponseStatus(HttpStatus.NOT_FOUND)` | Set the default HTTP status for an exception |

```java
@ResponseStatus(HttpStatus.NOT_FOUND)   // Any time this is thrown → 404 automatically
public class OrderNotFoundException extends RuntimeException {
    public OrderNotFoundException(Long id) {
        super("Order not found: " + id);
    }
}
```

---

## 7.11 Testing Annotations

| Annotation | What It Does |
|-----------|-------------|
| `@SpringBootTest` | Loads full application context for integration tests |
| `@WebMvcTest(Controller.class)` | Loads only the web layer (fast controller tests) |
| `@DataJpaTest` | Loads only JPA components (fast repository tests) |
| `@MockBean` | Replace a bean with a Mockito mock in the test context |
| `@Test` | Marks a test method (JUnit 5) |
| `@BeforeEach` / `@AfterEach` | Setup / teardown for each test |

```java
@WebMvcTest(OrderController.class)
class OrderControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean               // Replaces real OrderService with a mock
    private OrderService orderService;

    @Test
    void shouldReturnOrder() throws Exception {
        when(orderService.getOrderById(1L)).thenReturn(new Order("Laptop", 999.99));

        mockMvc.perform(get("/api/orders/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.product").value("Laptop"));
    }
}
```

---

## 7.12 Quick Reference — All Annotations at a Glance

| Category | Annotations |
|----------|------------|
| **App Setup** | `@SpringBootApplication`, `@Configuration`, `@ComponentScan` |
| **Bean Registration** | `@Component`, `@Service`, `@Repository`, `@Controller`, `@RestController`, `@Bean` |
| **Injection** | `@Autowired`, `@Qualifier`, `@Primary`, `@Value`, `@Lazy` |
| **Web / REST** | `@RequestMapping`, `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping` |
| **Parameters** | `@PathVariable`, `@RequestParam`, `@RequestBody`, `@RequestHeader` |
| **Validation** | `@Valid`, `@NotNull`, `@NotBlank`, `@Size`, `@Min`, `@Max`, `@Email` |
| **JPA Entity** | `@Entity`, `@Table`, `@Id`, `@GeneratedValue`, `@Column`, `@Transient` |
| **JPA Relations** | `@OneToMany`, `@ManyToOne`, `@OneToOne`, `@ManyToMany`, `@JoinColumn` |
| **Transaction** | `@Transactional`, `@Modifying` |
| **Caching** | `@Cacheable`, `@CacheEvict`, `@CachePut` |
| **Profiles** | `@Profile`, `@ConditionalOnProperty` |
| **Scheduling** | `@Scheduled`, `@Async`, `@EnableScheduling`, `@EnableAsync` |
| **Exceptions** | `@ControllerAdvice`, `@ExceptionHandler`, `@ResponseStatus` |
| **Testing** | `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest`, `@MockBean` |

---

### Interview Tip 🗣️

> "Spring Boot annotations are metadata tags that tell the framework what to do. `@Component` family registers beans, `@Autowired` wires them, `@GetMapping` maps URLs, `@Transactional` manages DB transactions, and `@ControllerAdvice` handles errors globally. The key is understanding that **Spring reads these at startup** to auto-configure your entire application."

---

[⬅️ Back to Curriculum](./README.md) | [⬅️ Previous: DTO & Mapping](./06-dto-and-mapping.md) | [Next: REST API Best Practices ➡️](./08-rest-api-best-practices.md)
