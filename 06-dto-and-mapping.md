# Module 6: DTO (Data Transfer Object) & Mapping

---

## 6.1 What is a DTO?

A DTO is a **separate Java class** used specifically for transferring data between the client and the server. It acts as a **filter** between your internal Entity (database model) and what the outside world sees.

### Layman Example: Your Resume vs. Your Diary 📝

```
Your DIARY (Entity)                    Your RESUME (DTO)
────────────────────                   ─────────────────
- Full name                            - Full name
- Date of birth                        - Date of birth
- Address                              - City (not full address)
- Bank account number      ❌ HIDDEN   
- Medical history          ❌ HIDDEN   
- Embarrassing secrets     ❌ HIDDEN   
- Work experience                      - Work experience
- Skills                               - Skills
- Internal employee ID     ❌ HIDDEN   
- Database password hash   ❌ HIDDEN   
- Salary                   ❌ HIDDEN   
```

You'd **never** hand your diary to a stranger. You give them a **resume** — a filtered, safe, formatted version of your information.

That's what a DTO does. It's the **resume** you show to the API consumer.

---

## 6.2 Why Not Just Use the Entity Directly?

```java
// ❌ BAD — Exposing Entity directly in the API
@GetMapping("/{id}")
public ResponseEntity<Employee> getEmployee(@PathVariable Long id) {
    return ResponseEntity.ok(employeeRepository.findById(id).get());
    // This returns EVERYTHING: password hash, internal IDs, lazy-loaded collections...
}
```

### Problems with Exposing Entities

| Problem | Example |
|---------|---------|
| **Security risk** | Exposes `passwordHash`, `internalNotes`, `salary` to the client |
| **Over-fetching** | Client gets 20 fields when they only need 3 |
| **Tight coupling** | If you rename a DB column, your API contract breaks |
| **Circular references** | Employee → Department → List\<Employee\> → Department → 💥 infinite loop (JSON serialization crash) |
| **Validation mixing** | Entity has DB annotations (`@Column`), API input needs different validation (`@NotBlank`) |
| **Different views** | Create endpoint needs `name + email`, Response needs `id + name + createdAt` — one class can't serve both |

---

## 6.3 The Solution — Separate DTOs

### The Pattern

```
Client sends JSON → Request DTO → Controller → Service → Entity → Repository → Database
Database → Entity → Service → Response DTO → Controller → JSON sent to Client
```

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│  Request DTO │  ──→    │    Entity     │  ──→    │ Response DTO │
│  (what comes │  map    │  (what's in   │  map    │ (what goes   │
│   IN from    │         │   the DB)     │         │  OUT to the  │
│   client)    │         │               │         │   client)    │
└──────────────┘         └──────────────┘         └──────────────┘
```

### Example: Employee System

#### The Entity (internal — maps to DB)

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
    private String passwordHash;        // ⚠️ NEVER expose this
    private Double salary;              // ⚠️ Sensitive
    private String internalNotes;       // ⚠️ Internal only
    private String department;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private boolean active;

    // Getters, setters, constructors...
}
```

#### Request DTO (what the client SENDS to create/update)

```java
import jakarta.validation.constraints.*;

public class EmployeeCreateRequest {

    @NotBlank(message = "First name is required")
    private String firstName;

    @NotBlank(message = "Last name is required")
    private String lastName;

    @Email(message = "Invalid email format")
    @NotBlank(message = "Email is required")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;            // Plain text password (will be hashed in service)

    @NotBlank(message = "Department is required")
    private String department;

    // Getters and setters
    // NO id, NO salary, NO createdAt — client doesn't set these
}
```

#### Response DTO (what the client RECEIVES)

```java
public class EmployeeResponse {

    private Long id;
    private String firstName;
    private String lastName;
    private String fullName;            // Computed field: firstName + " " + lastName
    private String email;
    private String department;
    private boolean active;
    private LocalDateTime createdAt;

    // NO passwordHash — never expose!
    // NO salary — sensitive!
    // NO internalNotes — internal only!
    // NO updatedAt — client doesn't need it

    // Getters and setters
    // Or use a constructor / builder
}
```

#### Update DTO (different from create — password not required)

```java
public class EmployeeUpdateRequest {

    @NotBlank(message = "First name is required")
    private String firstName;

    @NotBlank(message = "Last name is required")
    private String lastName;

    @Email(message = "Invalid email format")
    private String email;

    private String department;

    // NO password — update is separate
    // NO id — comes from the URL path

    // Getters and setters
}
```

---

## 6.4 Mapping — Converting Between Entity and DTO

### Approach 1: Manual Mapping (Simple & Explicit)

```java
@Service
public class EmployeeService {

    private final EmployeeRepository employeeRepository;

    public EmployeeService(EmployeeRepository employeeRepository) {
        this.employeeRepository = employeeRepository;
    }

    // ─── CREATE: Request DTO → Entity → Response DTO ────
    public EmployeeResponse createEmployee(EmployeeCreateRequest request) {

        // Map Request DTO → Entity
        Employee employee = new Employee();
        employee.setFirstName(request.getFirstName());
        employee.setLastName(request.getLastName());
        employee.setEmail(request.getEmail());
        employee.setPasswordHash(hashPassword(request.getPassword()));  // Hash it!
        employee.setDepartment(request.getDepartment());
        employee.setActive(true);
        employee.setCreatedAt(LocalDateTime.now());

        Employee saved = employeeRepository.save(employee);

        // Map Entity → Response DTO
        return mapToResponse(saved);
    }

    // ─── READ: Entity → Response DTO ────────────────────
    public EmployeeResponse getEmployee(Long id) {
        Employee employee = employeeRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Employee", "id", id));

        return mapToResponse(employee);
    }

    // ─── Reusable mapping method ────────────────────────
    private EmployeeResponse mapToResponse(Employee entity) {
        EmployeeResponse response = new EmployeeResponse();
        response.setId(entity.getId());
        response.setFirstName(entity.getFirstName());
        response.setLastName(entity.getLastName());
        response.setFullName(entity.getFirstName() + " " + entity.getLastName());
        response.setEmail(entity.getEmail());
        response.setDepartment(entity.getDepartment());
        response.setActive(entity.isActive());
        response.setCreatedAt(entity.getCreatedAt());
        // passwordHash, salary, internalNotes are NOT mapped — they stay hidden
        return response;
    }
}
```

### Approach 2: Using a Separate Mapper Class (Better Separation)

```java
@Component
public class EmployeeMapper {

    public Employee toEntity(EmployeeCreateRequest request) {
        Employee employee = new Employee();
        employee.setFirstName(request.getFirstName());
        employee.setLastName(request.getLastName());
        employee.setEmail(request.getEmail());
        employee.setDepartment(request.getDepartment());
        employee.setActive(true);
        employee.setCreatedAt(LocalDateTime.now());
        return employee;
    }

    public EmployeeResponse toResponse(Employee entity) {
        EmployeeResponse response = new EmployeeResponse();
        response.setId(entity.getId());
        response.setFirstName(entity.getFirstName());
        response.setLastName(entity.getLastName());
        response.setFullName(entity.getFirstName() + " " + entity.getLastName());
        response.setEmail(entity.getEmail());
        response.setDepartment(entity.getDepartment());
        response.setActive(entity.isActive());
        response.setCreatedAt(entity.getCreatedAt());
        return response;
    }

    public List<EmployeeResponse> toResponseList(List<Employee> entities) {
        return entities.stream()
                       .map(this::toResponse)
                       .collect(Collectors.toList());
    }

    public void updateEntity(Employee entity, EmployeeUpdateRequest request) {
        entity.setFirstName(request.getFirstName());
        entity.setLastName(request.getLastName());
        if (request.getEmail() != null) entity.setEmail(request.getEmail());
        if (request.getDepartment() != null) entity.setDepartment(request.getDepartment());
        entity.setUpdatedAt(LocalDateTime.now());
    }
}
```

**Usage in Service:**

```java
@Service
public class EmployeeService {

    private final EmployeeRepository employeeRepository;
    private final EmployeeMapper employeeMapper;

    public EmployeeService(EmployeeRepository employeeRepository, EmployeeMapper employeeMapper) {
        this.employeeRepository = employeeRepository;
        this.employeeMapper = employeeMapper;
    }

    public EmployeeResponse createEmployee(EmployeeCreateRequest request) {
        Employee employee = employeeMapper.toEntity(request);
        Employee saved = employeeRepository.save(employee);
        return employeeMapper.toResponse(saved);
    }

    public List<EmployeeResponse> getAllEmployees() {
        return employeeMapper.toResponseList(employeeRepository.findAll());
    }
}
```

### Approach 3: MapStruct (Auto-Generated Mapping — Production Grade)

MapStruct generates mapping code at **compile time** — zero runtime overhead.

```java
// Add to pom.xml
// <dependency>
//     <groupId>org.mapstruct</groupId>
//     <artifactId>mapstruct</artifactId>
//     <version>1.5.5.Final</version>
// </dependency>

@Mapper(componentModel = "spring")  // Makes it a Spring bean
public interface EmployeeMapper {

    // MapStruct auto-maps fields with matching names
    Employee toEntity(EmployeeCreateRequest request);

    @Mapping(target = "fullName", expression = "java(entity.getFirstName() + \" \" + entity.getLastName())")
    EmployeeResponse toResponse(Employee entity);

    List<EmployeeResponse> toResponseList(List<Employee> entities);

    // For partial updates
    @BeanMapping(nullValuePropertyMappingStrategy = NullValuePropertyMappingStrategy.IGNORE)
    void updateEntity(@MappingTarget Employee entity, EmployeeUpdateRequest request);
}
```

MapStruct generates the implementation code at compile time — no reflection, no runtime cost.

### Which Approach to Use?

| Approach | When to Use |
|----------|-------------|
| **Manual mapping** | Small project, few DTOs, learning phase |
| **Mapper class** | Medium project, want clean separation |
| **MapStruct** | Large project, many DTOs, production code |

---

## 6.5 The Controller with DTOs

```java
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    private final EmployeeService employeeService;

    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }

    @PostMapping
    public ResponseEntity<EmployeeResponse> create(@Valid @RequestBody EmployeeCreateRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
                             .body(employeeService.createEmployee(request));
    }

    @GetMapping("/{id}")
    public ResponseEntity<EmployeeResponse> getById(@PathVariable Long id) {
        return ResponseEntity.ok(employeeService.getEmployee(id));
    }

    @GetMapping
    public ResponseEntity<List<EmployeeResponse>> getAll() {
        return ResponseEntity.ok(employeeService.getAllEmployees());
    }

    @PutMapping("/{id}")
    public ResponseEntity<EmployeeResponse> update(
            @PathVariable Long id,
            @Valid @RequestBody EmployeeUpdateRequest request) {
        return ResponseEntity.ok(employeeService.updateEmployee(id, request));
    }
}
```

**Notice:**
- Input = `EmployeeCreateRequest` or `EmployeeUpdateRequest`
- Output = `EmployeeResponse`
- `Employee` entity NEVER appears in the controller — it stays internal

---

## 6.6 Project Structure with DTOs

```
src/main/java/com/example/app/
├── controller/
│   └── EmployeeController.java
├── service/
│   └── EmployeeService.java
├── repository/
│   └── EmployeeRepository.java
├── entity/
│   └── Employee.java
├── dto/                              ← DTOs live here
│   ├── request/
│   │   ├── EmployeeCreateRequest.java
│   │   └── EmployeeUpdateRequest.java
│   └── response/
│       └── EmployeeResponse.java
├── mapper/                           ← Mappers live here
│   └── EmployeeMapper.java
└── exception/
    └── GlobalExceptionHandler.java
```

---

## 6.7 Common Interview Questions & Answers

**Q: Why use DTOs instead of exposing entities?**
> DTOs **decouple** the API contract from the database schema. They prevent sensitive data leakage, avoid circular reference issues, allow different input/output shapes, and let the API and database evolve independently.

**Q: Isn't having separate DTOs a lot of boilerplate?**
> Yes, but the trade-off is worth it. Use **MapStruct** to auto-generate mapping code. The boilerplate cost is small compared to the security, flexibility, and maintainability benefits.

**Q: Should the Service layer return DTOs or Entities?**
> Two schools of thought:
> - **Service returns Entity** → Controller or a mapper converts to DTO (keeps service pure)
> - **Service returns DTO** → Service handles the mapping (keeps controller thin)
> Both are valid. In most Spring Boot projects, **the service returns DTOs** — this keeps the entity from leaking into the controller layer.

**Q: What is the difference between a DTO and a VO (Value Object)?**
> A **DTO** is for data transfer between layers/systems — it's a data carrier. A **VO** (Value Object) is defined by its values, not its identity — two VOs with the same values are considered equal (e.g., `Money(100, "USD")`).

---

## 6.8 Quick Revision Table

| Concept | One-Liner |
|---------|-----------|
| DTO | A filtered data carrier — shows only what the client should see |
| Entity | Maps to the database — contains all fields, including sensitive ones |
| Request DTO | What the client sends IN (has validation annotations) |
| Response DTO | What the client gets OUT (has computed/safe fields) |
| Mapper | Converts between Entity and DTO |
| MapStruct | Library that auto-generates mapper code at compile time |
| Why DTOs? | Security, decoupling, different input/output shapes, no circular refs |

---

### Interview Tip 🗣️

> "I never expose entities directly in APIs. I use **Request DTOs** for input (with validation) and **Response DTOs** for output (hiding sensitive fields). This decouples the API contract from the database schema, so they can evolve independently. For mapping, I use MapStruct in production for zero-overhead code generation."

---

[⬅️ Back to Curriculum](./README.md) | [⬅️ Previous: Exception Handling](./05-exception-handling.md) | [Next: Spring Boot Annotations ➡️](./07-spring-boot-annotations.md)
