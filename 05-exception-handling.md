# Module 5: Exception Handling in Spring Boot

---

## 5.1 The Problem — Why Do We Need Global Exception Handling?

### Layman Example: A Hospital Reception 🏥

Imagine a hospital without a proper error handling system:

```
Patient walks in → "I need help!"
Doctor 1: "Not my department" → throws the patient out the door
Doctor 2: "I crashed" → shows the patient all the internal medical records by accident
Doctor 3: "Error 500" → patient has no idea what happened
```

**This is what happens without proper exception handling:**
- Your API throws ugly stack traces to the client
- Sensitive internal details leak out
- Every controller has repetitive try-catch blocks
- No consistent error response format

**Now imagine the hospital has a proper reception desk:**

```
Patient walks in → "I need help!"
Reception (Global Exception Handler):
  - "Doctor is unavailable" → "Sorry, please come back at 3 PM" (clean message)
  - "Invalid insurance" → "Your insurance card is expired, here's how to renew" (helpful message)
  - "System crash" → "We're experiencing issues, please wait" (hides internals)
```

That reception desk is `@ControllerAdvice` — a **global exception handler** for your entire application.

---

## 5.2 The Bad Way — Try-Catch Everywhere

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping("/{id}")
    public ResponseEntity<?> getOrder(@PathVariable Long id) {
        try {
            Order order = orderService.getOrderById(id);
            return ResponseEntity.ok(order);
        } catch (OrderNotFoundException e) {
            return ResponseEntity.status(404).body("Order not found");
        } catch (IllegalArgumentException e) {
            return ResponseEntity.status(400).body("Invalid input");
        } catch (Exception e) {
            return ResponseEntity.status(500).body("Something went wrong");
        }
    }

    @PostMapping
    public ResponseEntity<?> createOrder(@RequestBody Order order) {
        try {
            Order saved = orderService.placeOrder(order);
            return ResponseEntity.status(201).body(saved);
        } catch (IllegalArgumentException e) {
            return ResponseEntity.status(400).body("Invalid input");
        } catch (Exception e) {
            return ResponseEntity.status(500).body("Something went wrong");
        }
    }

    // Imagine 20 more endpoints... all with the SAME try-catch blocks 😩
}
```

**Problems:**
- **Code duplication** — same catch blocks in every endpoint
- **Inconsistent** — one developer returns `"Order not found"`, another returns `"Not found"`
- **Hard to maintain** — want to change the error format? Change it in 50 places
- **Controller is bloated** — error handling code is bigger than business logic

---

## 5.3 The Good Way — Global Exception Handling

### Step 1: Create Custom Exceptions

```java
// ─── Base exception for "resource not found" ───
public class ResourceNotFoundException extends RuntimeException {

    private final String resourceName;
    private final String fieldName;
    private final Object fieldValue;

    public ResourceNotFoundException(String resourceName, String fieldName, Object fieldValue) {
        super(String.format("%s not found with %s: '%s'", resourceName, fieldName, fieldValue));
        this.resourceName = resourceName;
        this.fieldName = fieldName;
        this.fieldValue = fieldValue;
    }

    public String getResourceName() { return resourceName; }
    public String getFieldName() { return fieldName; }
    public Object getFieldValue() { return fieldValue; }
}

// ─── Specific exceptions ───
public class OrderNotFoundException extends ResourceNotFoundException {
    public OrderNotFoundException(Long id) {
        super("Order", "id", id);
    }
}

public class EmployeeNotFoundException extends ResourceNotFoundException {
    public EmployeeNotFoundException(Long id) {
        super("Employee", "id", id);
    }
}

// ─── Business logic exception ───
public class BusinessException extends RuntimeException {
    public BusinessException(String message) {
        super(message);
    }
}
```

### Step 2: Create a Standard Error Response DTO

```java
import java.time.LocalDateTime;
import java.util.Map;

public class ErrorResponse {

    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private Map<String, String> validationErrors;  // for field-level errors

    // Constructor for simple errors
    public ErrorResponse(int status, String error, String message, String path) {
        this.timestamp = LocalDateTime.now();
        this.status = status;
        this.error = error;
        this.message = message;
        this.path = path;
    }

    // Constructor with validation errors
    public ErrorResponse(int status, String error, String message, String path,
                         Map<String, String> validationErrors) {
        this(status, error, message, path);
        this.validationErrors = validationErrors;
    }

    // Getters and setters...
    public LocalDateTime getTimestamp() { return timestamp; }
    public int getStatus() { return status; }
    public String getError() { return error; }
    public String getMessage() { return message; }
    public String getPath() { return path; }
    public Map<String, String> getValidationErrors() { return validationErrors; }
}
```

### Step 3: Create the Global Exception Handler

```java
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ControllerAdvice;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.context.request.WebRequest;
import java.util.HashMap;
import java.util.Map;

@ControllerAdvice  // This class handles exceptions for ALL controllers
public class GlobalExceptionHandler {

    // ─── Handle "Resource Not Found" (404) ──────────────────
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(
            ResourceNotFoundException ex, WebRequest request) {

        ErrorResponse error = new ErrorResponse(
                HttpStatus.NOT_FOUND.value(),
                "Not Found",
                ex.getMessage(),
                request.getDescription(false).replace("uri=", "")
        );

        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }

    // ─── Handle Business Logic Errors (400) ─────────────────
    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusinessException(
            BusinessException ex, WebRequest request) {

        ErrorResponse error = new ErrorResponse(
                HttpStatus.BAD_REQUEST.value(),
                "Bad Request",
                ex.getMessage(),
                request.getDescription(false).replace("uri=", "")
        );

        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    // ─── Handle Validation Errors (400) ─────────────────────
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationErrors(
            MethodArgumentNotValidException ex, WebRequest request) {

        Map<String, String> fieldErrors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(fieldError ->
                fieldErrors.put(fieldError.getField(), fieldError.getDefaultMessage())
        );

        ErrorResponse error = new ErrorResponse(
                HttpStatus.BAD_REQUEST.value(),
                "Validation Failed",
                "One or more fields have invalid values",
                request.getDescription(false).replace("uri=", ""),
                fieldErrors
        );

        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    // ─── Handle Illegal Arguments (400) ─────────────────────
    @ExceptionHandler(IllegalArgumentException.class)
    public ResponseEntity<ErrorResponse> handleIllegalArgument(
            IllegalArgumentException ex, WebRequest request) {

        ErrorResponse error = new ErrorResponse(
                HttpStatus.BAD_REQUEST.value(),
                "Bad Request",
                ex.getMessage(),
                request.getDescription(false).replace("uri=", "")
        );

        return new ResponseEntity<>(error, HttpStatus.BAD_REQUEST);
    }

    // ─── Handle EVERYTHING ELSE (500) — the safety net ──────
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAllOtherExceptions(
            Exception ex, WebRequest request) {

        // Log the actual error (for developers)
        // logger.error("Unexpected error", ex);

        ErrorResponse error = new ErrorResponse(
                HttpStatus.INTERNAL_SERVER_ERROR.value(),
                "Internal Server Error",
                "Something went wrong. Please try again later.",  // Don't expose internals!
                request.getDescription(false).replace("uri=", "")
        );

        return new ResponseEntity<>(error, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

### Step 4: Now Your Controllers Are Clean

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(@PathVariable Long id) {
        return ResponseEntity.ok(orderService.getOrderById(id));
        // If order not found, service throws OrderNotFoundException
        // GlobalExceptionHandler catches it → returns clean 404 JSON
    }

    @PostMapping
    public ResponseEntity<Order> createOrder(@Valid @RequestBody Order order) {
        return ResponseEntity.status(HttpStatus.CREATED)
                             .body(orderService.placeOrder(order));
        // If validation fails → GlobalExceptionHandler returns clean 400 JSON
        // If business rule fails → GlobalExceptionHandler returns clean 400 JSON
    }

    // NO try-catch anywhere! Clean and focused.
}
```

### What the Error Response Looks Like

```json
// 404 — Resource not found
{
    "timestamp": "2026-02-24T22:00:00",
    "status": 404,
    "error": "Not Found",
    "message": "Order not found with id: '42'",
    "path": "/api/orders/42"
}

// 400 — Validation error
{
    "timestamp": "2026-02-24T22:00:00",
    "status": 400,
    "error": "Validation Failed",
    "message": "One or more fields have invalid values",
    "path": "/api/orders",
    "validationErrors": {
        "customerName": "must not be blank",
        "amount": "must be greater than 0"
    }
}

// 500 — Server error (no internals exposed!)
{
    "timestamp": "2026-02-24T22:00:00",
    "status": 500,
    "error": "Internal Server Error",
    "message": "Something went wrong. Please try again later.",
    "path": "/api/orders"
}
```

---

## 5.4 Bean Validation (`@Valid`)

Spring Boot integrates with **Jakarta Bean Validation** to validate request bodies automatically.

### Add Validation to Your Entity/DTO

```java
import jakarta.validation.constraints.*;

public class OrderRequest {

    @NotBlank(message = "Customer name is required")
    private String customerName;

    @NotBlank(message = "Product is required")
    private String product;

    @NotNull(message = "Amount is required")
    @Min(value = 1, message = "Amount must be at least $1")
    @Max(value = 100000, message = "Amount cannot exceed $100,000")
    private Double amount;

    @Email(message = "Invalid email format")
    private String email;

    @Size(min = 10, max = 15, message = "Phone must be 10-15 characters")
    private String phone;

    // Getters and setters
}
```

### Common Validation Annotations

| Annotation | What It Checks |
|-----------|---------------|
| `@NotNull` | Field is not null |
| `@NotBlank` | String is not null, not empty, not just whitespace |
| `@NotEmpty` | String/Collection is not null and not empty |
| `@Size(min, max)` | String length or collection size is within range |
| `@Min(value)` | Number >= value |
| `@Max(value)` | Number <= value |
| `@Email` | Valid email format |
| `@Pattern(regexp)` | Matches a regex pattern |
| `@Past` | Date is in the past |
| `@Future` | Date is in the future |
| `@Positive` | Number > 0 |
| `@PositiveOrZero` | Number >= 0 |

### Trigger Validation in Controller

```java
@PostMapping
public ResponseEntity<Order> createOrder(@Valid @RequestBody OrderRequest request) {
    //                                    ^^^^^^
    //                                    This triggers validation!
    //                                    If validation fails → MethodArgumentNotValidException
    //                                    → caught by GlobalExceptionHandler

    return ResponseEntity.status(HttpStatus.CREATED)
                         .body(orderService.placeOrder(request));
}
```

---

## 5.5 Exception Hierarchy — Which to Extend?

```
Throwable
├── Error                ← JVM errors (OutOfMemoryError) — DON'T catch these
└── Exception
    ├── Checked Exceptions (must declare with 'throws' or try-catch)
    │   └── IOException, SQLException, etc.
    └── RuntimeException  ← UNCHECKED — preferred for Spring Boot
        ├── IllegalArgumentException
        ├── NullPointerException
        └── Your custom exceptions ← extend RuntimeException
```

**Rule:** In Spring Boot, extend `RuntimeException` for custom exceptions. This way:
- You don't need to declare `throws` everywhere
- Spring's `@ExceptionHandler` catches them automatically
- Code stays clean

---

## 5.6 Common Interview Questions & Answers

**Q: What is `@ControllerAdvice`?**
> It's a global exception handler that applies to ALL controllers. Any exception thrown in any controller is intercepted by `@ControllerAdvice` before reaching the client, allowing you to return a clean, consistent error response.

**Q: What is the difference between `@ControllerAdvice` and `@RestControllerAdvice`?**
> `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`. Use `@RestControllerAdvice` for REST APIs (returns JSON). Use `@ControllerAdvice` if you're returning views (HTML).

**Q: Why use custom exceptions instead of generic ones?**
> Custom exceptions carry **context** (resource name, field, value) and can be mapped to specific HTTP status codes. `RuntimeException("Order not found")` tells you nothing — `OrderNotFoundException(42)` tells you exactly what happened.

**Q: Where should validation logic live — Controller or Service?**
> **Both.** Use `@Valid` in the controller for **input format validation** (not blank, valid email). Use the service layer for **business rule validation** (minimum order amount, stock availability).

**Q: What happens if you have two `@ExceptionHandler` methods that match the same exception?**
> The **most specific** handler wins. If you have handlers for both `OrderNotFoundException` and `RuntimeException`, and an `OrderNotFoundException` is thrown, the `OrderNotFoundException` handler is used.

---

## 5.7 Quick Revision Table

| Concept | One-Liner |
|---------|-----------|
| `@ControllerAdvice` | Global exception handler for all controllers |
| `@ExceptionHandler` | Marks a method as a handler for a specific exception type |
| `@Valid` | Triggers bean validation on a request body |
| `@NotBlank` | Not null + not empty + not whitespace |
| `ErrorResponse` | A DTO that standardizes all error responses |
| Custom Exception | Extend `RuntimeException`, carry context, map to status codes |
| Checked vs Unchecked | Use unchecked (`RuntimeException`) in Spring Boot |

---

### Interview Tip 🗣️

> "I use `@ControllerAdvice` as a **global reception desk** that catches all exceptions and returns a consistent JSON error response. Controllers stay clean — no try-catch blocks. Custom exceptions carry context (what failed and why), and `@Valid` handles input validation automatically."

---

[⬅️ Back to Curriculum](./README.md) | [⬅️ Previous: JPA Repository](./04-jpa-repository.md) | [Next: DTO & Mapping ➡️](./06-dto-and-mapping.md)
