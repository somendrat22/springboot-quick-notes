# Module 4: JPA Repository

---

## 4.1 What is JPA?

**JPA (Java Persistence API)** is a **specification** — a set of rules that defines how Java objects should be saved to and loaded from a relational database.

**Hibernate** is the most popular **implementation** of that specification.

### Layman Analogy: Driving License 🚗

```
JPA       = The driving rules (specification)
            "You must stop at red lights, drive on the left, use indicators..."

Hibernate = A driver who follows those rules (implementation)
            An actual person who knows how to drive by those rules

Spring Data JPA = An Uber app
            You don't even need to drive — just tell the app where you want to go,
            and it handles everything (finding a driver, navigation, payment).
```

### The Stack

```
Your Code
    ↓ uses
Spring Data JPA          ← Makes JPA super easy (auto-generates queries from method names)
    ↓ uses
JPA (Specification)      ← The rules
    ↓ implemented by
Hibernate                ← The engine that actually talks to the DB
    ↓ uses
JDBC                     ← Low-level Java database connection
    ↓ connects to
Database (MySQL, PostgreSQL, etc.)
```

---

## 4.2 What is JpaRepository?

`JpaRepository` is an **interface** provided by Spring Data JPA. When you extend it, Spring **auto-generates a full implementation** at runtime — you write zero SQL, zero implementation code.

### Layman Analogy: A Vending Machine 🎰

Imagine a vending machine for your database:

```
┌─────────────────────────────────────────┐
│        DATABASE VENDING MACHINE          │
│                                          │
│   [A1] Save an item         → save()    │
│   [A2] Find by ID           → findById()│
│   [A3] Get all items        → findAll() │
│   [A4] Delete an item       → delete()  │
│   [A5] Count items          → count()   │
│   [A6] Check if exists      → existsById()│
│                                          │
│   ★ CUSTOM BUTTONS ★                    │
│   Just NAME your button correctly        │
│   and it auto-generates the SQL!         │
│                                          │
│   [B1] findByName("John")               │
│   [B2] findByAgeGreaterThan(25)          │
│   [B3] findByDepartmentAndSalary(...)    │
│                                          │
└─────────────────────────────────────────┘

You press buttons → machine gives you results.
You NEVER open the machine to see how it works.
```

---

## 4.3 The Repository Hierarchy

Spring Data JPA has a chain of interfaces, each adding more features:

```
Repository                              ← Marker interface (empty, just says "I'm a repository")
    │
    └── CrudRepository<T, ID>           ← Basic CRUD: save, findById, findAll, delete, count
            │
            └── ListCrudRepository<T,ID>  ← Same as CrudRepository but returns List instead of Iterable
                    │
                    └── PagingAndSortingRepository<T, ID>  ← Adds pagination & sorting
                            │
                            └── JpaRepository<T, ID>       ← Adds: flush, saveAllAndFlush, batch deletes
```

**Rule of thumb:** Always extend `JpaRepository` — it includes everything from the parents.

---

## 4.4 What You Get for FREE

Just by writing this:

```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    // THAT'S IT. No implementation. No SQL.
}
```

You automatically get **ALL of these methods**:

### Create / Update

| Method | SQL Equivalent | Notes |
|--------|---------------|-------|
| `save(entity)` | `INSERT` or `UPDATE` | If entity has no ID → INSERT. If it has an ID → UPDATE |
| `saveAll(List<Entity>)` | Batch INSERT/UPDATE | Save multiple entities at once |
| `saveAndFlush(entity)` | INSERT/UPDATE + immediate flush | Writes to DB immediately (not waiting for transaction end) |

### Read

| Method | SQL Equivalent |
|--------|---------------|
| `findById(id)` | `SELECT * FROM table WHERE id = ?` — returns `Optional<T>` |
| `findAll()` | `SELECT * FROM table` |
| `findAll(Sort sort)` | `SELECT * FROM table ORDER BY ...` |
| `findAll(Pageable pageable)` | `SELECT * FROM table LIMIT ? OFFSET ?` |
| `findAllById(List<ID> ids)` | `SELECT * FROM table WHERE id IN (?, ?, ?)` |
| `getById(id)` / `getReferenceById(id)` | Returns a **lazy proxy** — only hits DB when you access a field |

### Delete

| Method | SQL Equivalent |
|--------|---------------|
| `deleteById(id)` | `DELETE FROM table WHERE id = ?` |
| `delete(entity)` | `DELETE FROM table WHERE id = ?` |
| `deleteAll()` | `DELETE FROM table` (⚠️ dangerous!) |
| `deleteAllById(List<ID> ids)` | `DELETE FROM table WHERE id IN (?, ?, ?)` |
| `deleteAllInBatch()` | Single bulk DELETE (more efficient than deleteAll) |

### Utility

| Method | SQL Equivalent |
|--------|---------------|
| `count()` | `SELECT COUNT(*) FROM table` |
| `existsById(id)` | `SELECT COUNT(*) > 0 FROM table WHERE id = ?` |
| `flush()` | Force pending changes to be written to DB |

---

## 4.5 Query Derivation — The Magic of Method Names

This is the **killer feature** of Spring Data JPA. You write a method name following a convention, and Spring **generates the SQL automatically**.

### Layman Analogy: Asking Google 🔍

Think of it like asking Google a question in natural English:

```
You type:     "Find employees by department"
Google hears: "Search for employees WHERE department = ?"
Google runs:   SELECT * FROM employees WHERE department = ?

You type:     "Find employees by age greater than 25"
Google hears: "Search for employees WHERE age > 25"
Google runs:   SELECT * FROM employees WHERE age > 25
```

That's EXACTLY how Spring Data JPA works — your method name IS the query.

### How It Works — The Grammar

```
findBy + FieldName + Condition
```

Spring parses your method name like this:

```
findByDepartmentAndSalaryGreaterThan(String dept, Double salary)
│      │            │   │
│      │            │   └── Condition: GreaterThan → ">"
│      │            └────── Conjunction: And → "AND"
│      └─────────────────── Field: Department → "WHERE department = ?"
└────────────────────────── Action: findBy → "SELECT * FROM ..."
```

Generated SQL:
```sql
SELECT * FROM employees WHERE department = ? AND salary > ?
```

### Complete Examples

```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // ─── SIMPLE FINDERS ─────────────────────────────────────

    // SELECT * FROM employee WHERE name = ?
    List<Employee> findByName(String name);

    // SELECT * FROM employee WHERE email = ?
    Optional<Employee> findByEmail(String email);

    // SELECT * FROM employee WHERE department = ?
    List<Employee> findByDepartment(String department);


    // ─── COMPARISON ─────────────────────────────────────────

    // SELECT * FROM employee WHERE salary > ?
    List<Employee> findBySalaryGreaterThan(Double salary);

    // SELECT * FROM employee WHERE salary < ?
    List<Employee> findBySalaryLessThan(Double salary);

    // SELECT * FROM employee WHERE salary BETWEEN ? AND ?
    List<Employee> findBySalaryBetween(Double min, Double max);

    // SELECT * FROM employee WHERE age >= ?
    List<Employee> findByAgeGreaterThanEqual(Integer age);


    // ─── STRING MATCHING ────────────────────────────────────

    // SELECT * FROM employee WHERE name LIKE '%keyword%'
    List<Employee> findByNameContaining(String keyword);

    // SELECT * FROM employee WHERE name LIKE 'prefix%'
    List<Employee> findByNameStartingWith(String prefix);

    // SELECT * FROM employee WHERE name LIKE '%suffix'
    List<Employee> findByNameEndingWith(String suffix);

    // SELECT * FROM employee WHERE UPPER(name) = UPPER(?)
    List<Employee> findByNameIgnoreCase(String name);


    // ─── BOOLEAN / NULL ─────────────────────────────────────

    // SELECT * FROM employee WHERE active = true
    List<Employee> findByActiveTrue();

    // SELECT * FROM employee WHERE active = false
    List<Employee> findByActiveFalse();

    // SELECT * FROM employee WHERE phone IS NULL
    List<Employee> findByPhoneIsNull();

    // SELECT * FROM employee WHERE phone IS NOT NULL
    List<Employee> findByPhoneIsNotNull();


    // ─── COMBINING CONDITIONS ───────────────────────────────

    // SELECT * FROM employee WHERE department = ? AND salary > ?
    List<Employee> findByDepartmentAndSalaryGreaterThan(String dept, Double salary);

    // SELECT * FROM employee WHERE department = ? OR salary > ?
    List<Employee> findByDepartmentOrSalaryGreaterThan(String dept, Double salary);

    // SELECT * FROM employee WHERE name = ? AND department = ? AND active = true
    List<Employee> findByNameAndDepartmentAndActiveTrue(String name, String dept);


    // ─── ORDERING ───────────────────────────────────────────

    // SELECT * FROM employee ORDER BY salary DESC
    List<Employee> findAllByOrderBySalaryDesc();

    // SELECT * FROM employee WHERE department = ? ORDER BY name ASC
    List<Employee> findByDepartmentOrderByNameAsc(String department);

    // SELECT * FROM employee WHERE department = ? ORDER BY salary DESC
    List<Employee> findByDepartmentOrderBySalaryDesc(String department);


    // ─── LIMITING RESULTS ───────────────────────────────────

    // SELECT * FROM employee ORDER BY salary DESC LIMIT 1
    Optional<Employee> findFirstByOrderBySalaryDesc();

    // SELECT * FROM employee ORDER BY salary DESC LIMIT 5
    List<Employee> findTop5ByOrderBySalaryDesc();

    // SELECT * FROM employee WHERE department = ? LIMIT 1
    Optional<Employee> findFirstByDepartment(String department);


    // ─── COLLECTION PARAMETERS ──────────────────────────────

    // SELECT * FROM employee WHERE department IN (?, ?, ?)
    List<Employee> findByDepartmentIn(List<String> departments);

    // SELECT * FROM employee WHERE department NOT IN (?, ?, ?)
    List<Employee> findByDepartmentNotIn(List<String> departments);


    // ─── COUNT & EXISTS & DELETE ─────────────────────────────

    // SELECT COUNT(*) FROM employee WHERE department = ?
    Long countByDepartment(String department);

    // SELECT COUNT(*) > 0 FROM employee WHERE email = ?
    boolean existsByEmail(String email);

    // DELETE FROM employee WHERE department = ?
    void deleteByDepartment(String department);
}
```

---

## 4.6 Keyword Cheat Sheet

| Keyword | SQL | Example Method | Generated SQL Fragment |
|---------|-----|---------------|----------------------|
| `And` | `AND` | `findByNameAndAge(n, a)` | `WHERE name=? AND age=?` |
| `Or` | `OR` | `findByNameOrAge(n, a)` | `WHERE name=? OR age=?` |
| `Between` | `BETWEEN` | `findByAgeBetween(a, b)` | `WHERE age BETWEEN ? AND ?` |
| `LessThan` | `<` | `findByAgeLessThan(a)` | `WHERE age < ?` |
| `LessThanEqual` | `<=` | `findByAgeLessThanEqual(a)` | `WHERE age <= ?` |
| `GreaterThan` | `>` | `findByAgeGreaterThan(a)` | `WHERE age > ?` |
| `GreaterThanEqual` | `>=` | `findByAgeGreaterThanEqual(a)` | `WHERE age >= ?` |
| `IsNull` | `IS NULL` | `findByPhoneIsNull()` | `WHERE phone IS NULL` |
| `IsNotNull` | `IS NOT NULL` | `findByPhoneIsNotNull()` | `WHERE phone IS NOT NULL` |
| `Like` | `LIKE` | `findByNameLike(p)` | `WHERE name LIKE ?` (you provide `%`) |
| `Containing` | `LIKE %…%` | `findByNameContaining(s)` | `WHERE name LIKE '%s%'` |
| `StartingWith` | `LIKE …%` | `findByNameStartingWith(s)` | `WHERE name LIKE 's%'` |
| `EndingWith` | `LIKE %…` | `findByNameEndingWith(s)` | `WHERE name LIKE '%s'` |
| `Not` | `<>` | `findByNameNot(n)` | `WHERE name <> ?` |
| `In` | `IN` | `findByAgeIn(list)` | `WHERE age IN (?,?,?)` |
| `NotIn` | `NOT IN` | `findByAgeNotIn(list)` | `WHERE age NOT IN (?,?,?)` |
| `True` | `= TRUE` | `findByActiveTrue()` | `WHERE active = TRUE` |
| `False` | `= FALSE` | `findByActiveFalse()` | `WHERE active = FALSE` |
| `OrderBy` | `ORDER BY` | `findByDeptOrderBySalaryDesc()` | `ORDER BY salary DESC` |
| `IgnoreCase` | `UPPER()` | `findByNameIgnoreCase(n)` | `WHERE UPPER(name)=UPPER(?)` |
| `Top` / `First` | `LIMIT` | `findTop5ByOrderBySalaryDesc()` | `ORDER BY salary DESC LIMIT 5` |

---

## 4.7 Custom Queries with @Query

When method names get too long or the query is too complex, use `@Query`:

### JPQL (Java Persistence Query Language)

JPQL uses **entity/field names** (Java names), NOT table/column names.

```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // Simple JPQL
    @Query("SELECT e FROM Employee e WHERE e.department = :dept")
    List<Employee> findByDept(@Param("dept") String department);

    // JPQL with multiple conditions
    @Query("SELECT e FROM Employee e WHERE e.department = :dept AND e.salary > :minSalary")
    List<Employee> findHighEarners(@Param("dept") String dept, @Param("minSalary") Double minSalary);

    // JPQL with LIKE
    @Query("SELECT e FROM Employee e WHERE e.name LIKE %:keyword%")
    List<Employee> searchByName(@Param("keyword") String keyword);

    // JPQL returning specific fields (projection)
    @Query("SELECT e.name, e.salary FROM Employee e WHERE e.department = :dept")
    List<Object[]> findNameAndSalaryByDept(@Param("dept") String department);

    // JPQL with JOIN
    @Query("SELECT e FROM Employee e JOIN e.department d WHERE d.name = :deptName")
    List<Employee> findByDepartmentName(@Param("deptName") String deptName);
}
```

### Native SQL

Use actual **table/column names**. Useful for complex DB-specific queries.

```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // Native SQL
    @Query(value = "SELECT * FROM employees WHERE department = ?1 AND salary > ?2",
           nativeQuery = true)
    List<Employee> findHighEarnersNative(String dept, Double minSalary);

    // Native SQL with named params
    @Query(value = "SELECT * FROM employees WHERE YEAR(hire_date) = :year",
           nativeQuery = true)
    List<Employee> findByHireYear(@Param("year") int year);

    // Native SQL for complex queries
    @Query(value = """
            SELECT d.name AS department, AVG(e.salary) AS avg_salary
            FROM employees e
            JOIN departments d ON e.department_id = d.id
            GROUP BY d.name
            HAVING AVG(e.salary) > :threshold
            """,
           nativeQuery = true)
    List<Object[]> findHighPayingDepartments(@Param("threshold") Double threshold);
}
```

### @Modifying — For UPDATE and DELETE Queries

```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // Update query — MUST have @Modifying and @Transactional
    @Modifying
    @Transactional
    @Query("UPDATE Employee e SET e.salary = e.salary * 1.10 WHERE e.department = :dept")
    int giveRaise(@Param("dept") String department);  // returns number of rows affected

    // Delete query
    @Modifying
    @Transactional
    @Query("DELETE FROM Employee e WHERE e.active = false")
    int deleteInactiveEmployees();  // returns number of rows deleted

    // Bulk update
    @Modifying
    @Transactional
    @Query("UPDATE Employee e SET e.department = :newDept WHERE e.department = :oldDept")
    int transferDepartment(@Param("oldDept") String oldDept, @Param("newDept") String newDept);
}
```

### JPQL vs Native SQL — When to Use What

| Feature | JPQL | Native SQL |
|---------|------|-----------|
| **Uses** | Entity & field names (`Employee`, `salary`) | Table & column names (`employees`, `salary_amount`) |
| **Database portable** | Yes — works on MySQL, PostgreSQL, Oracle | No — tied to specific DB syntax |
| **Complex queries** | Limited (no window functions, CTEs) | Full SQL power |
| **When to use** | 90% of the time | DB-specific features, complex reports, performance optimization |

---

## 4.8 Pagination & Sorting

### Layman Analogy: Amazon Search Results

When you search on Amazon, you don't see all 50,000 results at once. You see:
- **Page 1** of 5,000 pages
- **10 results** per page
- Sorted by **"Price: Low to High"**

That's pagination and sorting.

### In Code

**No extra repository method needed** — `findAll(Pageable)` is inherited from JpaRepository.

#### Service Layer

```java
@Service
public class EmployeeService {

    private final EmployeeRepository employeeRepository;

    public EmployeeService(EmployeeRepository employeeRepository) {
        this.employeeRepository = employeeRepository;
    }

    // ─── Pagination + Sorting ───
    public Page<Employee> getEmployees(int pageNo, int pageSize, String sortBy, String direction) {
        Sort sort = direction.equalsIgnoreCase("desc")
                ? Sort.by(sortBy).descending()
                : Sort.by(sortBy).ascending();

        Pageable pageable = PageRequest.of(pageNo, pageSize, sort);
        return employeeRepository.findAll(pageable);
    }

    // ─── Sorting only ───
    public List<Employee> getAllSorted(String sortBy) {
        return employeeRepository.findAll(Sort.by(sortBy).ascending());
    }

    // ─── Multi-field sorting ───
    public List<Employee> getAllSortedMultiple() {
        Sort sort = Sort.by(
            Sort.Order.asc("department"),
            Sort.Order.desc("salary")
        );
        return employeeRepository.findAll(sort);
    }
}
```

#### Controller Layer

```java
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    private final EmployeeService employeeService;

    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }

    // GET /api/employees?page=0&size=10&sortBy=salary&direction=desc
    @GetMapping
    public ResponseEntity<Page<Employee>> getEmployees(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "id") String sortBy,
            @RequestParam(defaultValue = "asc") String direction) {

        Page<Employee> result = employeeService.getEmployees(page, size, sortBy, direction);
        return ResponseEntity.ok(result);
    }
}
```

#### What the Response Looks Like

```json
{
    "content": [
        { "id": 1, "name": "Alice", "salary": 95000 },
        { "id": 2, "name": "Bob", "salary": 88000 }
    ],
    "pageable": {
        "pageNumber": 0,
        "pageSize": 10,
        "sort": { "sorted": true, "unsorted": false }
    },
    "totalElements": 150,
    "totalPages": 15,
    "number": 0,
    "size": 10,
    "first": true,
    "last": false,
    "empty": false
}
```

| Field | Meaning |
|-------|---------|
| `content` | The actual data for this page |
| `totalElements` | Total records in the database |
| `totalPages` | Total number of pages |
| `number` | Current page number (0-indexed) |
| `size` | Number of records per page |
| `first` | Is this the first page? |
| `last` | Is this the last page? |

### Custom Queries with Pagination

```java
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // Method-name derived query with pagination
    Page<Employee> findByDepartment(String department, Pageable pageable);

    // @Query with pagination
    @Query("SELECT e FROM Employee e WHERE e.salary > :minSalary")
    Page<Employee> findHighEarners(@Param("minSalary") Double minSalary, Pageable pageable);
}
```

---

## 4.9 Entity Relationships (Bonus — Often Asked)

### Layman Analogy

```
One Department has MANY Employees    → One-to-Many
One Employee belongs to ONE Department → Many-to-One
One Employee has ONE Address         → One-to-One
Many Students take MANY Courses      → Many-to-Many
```

### One-to-Many / Many-to-One

```java
@Entity
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL)
    private List<Employee> employees = new ArrayList<>();
}

@Entity
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @ManyToOne(fetch = FetchType.LAZY)      // LAZY = don't load department until accessed
    @JoinColumn(name = "department_id")     // FK column in employee table
    private Department department;
}
```

### Fetch Types

| Type | Behavior | When to Use |
|------|----------|-------------|
| `LAZY` (default for collections) | Load only when accessed | Most of the time — avoids N+1 |
| `EAGER` (default for single entities) | Load immediately with parent | Only when you ALWAYS need the related data |

### The N+1 Problem (VERY common interview question)

```
Problem:
    1 query to fetch 100 departments
    + 100 queries to fetch employees for each department
    = 101 queries! 😱

Solution: Use JOIN FETCH in JPQL
    @Query("SELECT d FROM Department d JOIN FETCH d.employees")
    List<Department> findAllWithEmployees();
    → Now it's just 1 query with a JOIN
```

---

## 4.10 Common Interview Questions & Answers

**Q: What is the difference between `findById()` and `getReferenceById()`?**
> `findById()` executes a SELECT immediately and returns `Optional<T>`.
> `getReferenceById()` returns a **lazy proxy** — the DB query only runs when you access a field. Throws `EntityNotFoundException` if the entity doesn't exist.

**Q: What is the difference between `save()` for insert vs update?**
> Spring checks if the entity has an ID.
> - **No ID (or ID is null)** → `INSERT` (new entity)
> - **Has an ID** → `UPDATE` (existing entity — does a `SELECT` first to check existence)

**Q: What is the difference between JPQL and native SQL?**
> JPQL uses entity/field names and is database-portable. Native SQL uses table/column names and can leverage DB-specific features. Use JPQL by default; use native SQL for complex queries.

**Q: What is `@Transactional` and where should it go?**
> It marks a method as a database transaction (all-or-nothing). It should go on the **Service layer**, not the Repository or Controller.

**Q: How do you handle the N+1 problem?**
> Use `JOIN FETCH` in JPQL, or use `@EntityGraph`, or configure batch fetching with `@BatchSize`. The key is to fetch related entities in as few queries as possible.

**Q: What is `spring.jpa.open-in-view` and should it be enabled?**
> It keeps the Hibernate session open until the HTTP response is sent, allowing lazy loading in the Controller/View layer. It should be **disabled in production** (`false`) because it can cause unexpected queries and performance issues. Handle all data loading in the Service layer instead.

---

## 4.11 Quick Revision Table

| Concept | One-Liner |
|---------|-----------|
| JPA | A specification for ORM (mapping Java objects to DB tables) |
| Hibernate | The most popular JPA implementation |
| Spring Data JPA | A wrapper that makes JPA effortless (auto-generates repos) |
| `JpaRepository` | Extend it → get all CRUD + paging + sorting for free |
| Query Derivation | Name your method correctly → Spring writes the SQL |
| `@Query` (JPQL) | Write queries using entity/field names (portable) |
| `@Query` (native) | Write raw SQL (DB-specific, full power) |
| `@Modifying` | Required for UPDATE/DELETE queries |
| `Pageable` | Pass it to any find method → get paginated results |
| `LAZY` fetch | Load related data only when accessed (preferred) |
| `EAGER` fetch | Load related data immediately (use sparingly) |
| N+1 Problem | 1 query + N extra queries; fix with JOIN FETCH |
| `save()` | No ID → INSERT; Has ID → UPDATE |

---

### Interview Tip 🗣️

> "JpaRepository is like a **vending machine for database operations**. You press buttons (write method names), and it gives you the result. You don't need to know how the machine works internally (SQL generation) — just follow the naming convention. For anything the naming convention can't handle, use `@Query` with JPQL or native SQL."

---

[⬅️ Back to Curriculum](./README.md) | [⬅️ Previous: Controller-Service-Repository](./03-controller-service-repository.md)
