# Module 9: Spring Security Basics

---

## 9.1 What is Spring Security?

Spring Security is a **framework that handles authentication and authorization** for your Spring Boot application. It protects your APIs from unauthorized access.

### Layman Example: A Nightclub 🎶

```
You arrive at a nightclub:

STEP 1: AUTHENTICATION (WHO are you?)
─────────────────────────────────────
  Bouncer: "Show me your ID."
  You:     *shows driver's license*
  Bouncer: "OK, you are John Doe, age 25. You're verified."

  → This is AUTHENTICATION: proving your identity.

STEP 2: AUTHORIZATION (WHAT can you do?)
────────────────────────────────────────
  Bouncer: "You have a GENERAL ticket. You can enter the main floor."
  Bouncer: "But the VIP lounge? No. You need a VIP ticket."

  → This is AUTHORIZATION: what you're allowed to do based on your role.

STEP 3: THROUGHOUT THE NIGHT
─────────────────────────────
  You wear a WRISTBAND (token) so you don't have to show your ID again.
  Every time you go to the bar, the bartender checks your wristband.

  → This is a SESSION/TOKEN: proof that you've already been authenticated.
```

---

## 9.2 Authentication vs Authorization

| Concept | Question It Answers | Nightclub Analogy | HTTP Code on Failure |
|---------|--------------------|--------------------|---------------------|
| **Authentication** | **WHO** are you? | Showing your ID to the bouncer | `401 Unauthorized` |
| **Authorization** | **WHAT** can you do? | Checking if your ticket is VIP or General | `403 Forbidden` |

**Order matters:** Authentication ALWAYS comes first. You can't check permissions if you don't know who the person is.

```
Request arrives
    │
    ▼
┌─────────────────────┐     NO      ┌──────────────┐
│ Is user authenticated? │ ─────────→ │ 401 Unauthorized │
└─────────┬───────────┘              └──────────────┘
          │ YES
          ▼
┌─────────────────────┐     NO      ┌──────────────┐
│ Is user authorized?   │ ─────────→ │ 403 Forbidden  │
└─────────┬───────────┘              └──────────────┘
          │ YES
          ▼
    ┌──────────────┐
    │ Process Request │
    └──────────────┘
```

---

## 9.3 How Authentication Works — The Big Picture

### Common Authentication Methods

| Method | How It Works | When to Use |
|--------|-------------|-------------|
| **Basic Auth** | Send username:password (Base64 encoded) in every request | Internal tools, quick prototypes |
| **Session-Based** | Server stores session; client sends session cookie | Traditional web apps (MVC + Thymeleaf) |
| **Token-Based (JWT)** | Client sends a signed token in every request | REST APIs, microservices, SPAs |
| **OAuth 2.0** | Delegate auth to a third party (Google, GitHub) | "Login with Google" flows |

**For REST APIs, JWT is the most common** — that's what we'll focus on.

---

## 9.4 JWT (JSON Web Token) — Explained Simply

### Layman Analogy: A Movie Ticket 🎬

```
1. You go to the ticket counter (LOGIN endpoint)
2. You show your ID and pay (send username + password)
3. Counter gives you a TICKET (JWT token)
4. The ticket has:
   - Your name (claims)
   - Seat number / movie (roles/permissions)
   - Expiry time (the movie ends at 9 PM)
   - A HOLOGRAM (signature) — proves it's genuine, not a fake

5. You show the ticket at the theater entrance (send token with each request)
6. The usher VERIFIES the hologram (verifies the JWT signature)
7. The usher checks the time (is the token expired?)
8. If valid → you enter. If not → you're turned away.

KEY INSIGHT: The usher never calls the ticket counter to verify.
The hologram (signature) is enough. This is why JWT is STATELESS.
```

### JWT Structure

A JWT has 3 parts separated by dots:

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJqb2huIiwicm9sZSI6IkFETUlOIiwiZXhwIjoxNzA5MzI1NTc4fQ.abc123signature

│         HEADER          │              PAYLOAD              │     SIGNATURE      │
```

**Decoded:**

```json
// HEADER — algorithm used
{
    "alg": "HS256",
    "typ": "JWT"
}

// PAYLOAD — the actual data (claims)
{
    "sub": "john",              // subject (username)
    "role": "ADMIN",            // custom claim
    "iat": 1709325578,          // issued at
    "exp": 1709329178           // expires at (1 hour later)
}

// SIGNATURE — ensures the token hasn't been tampered with
HMACSHA256(
    base64(header) + "." + base64(payload),
    your-secret-key
)
```

### JWT Flow

```
┌────────┐                          ┌────────┐
│ Client │                          │ Server │
└───┬────┘                          └───┬────┘
    │                                   │
    │  1. POST /api/auth/login          │
    │     { "username": "john",         │
    │       "password": "secret" }      │
    │ ─────────────────────────────────→│
    │                                   │  2. Validate credentials
    │                                   │  3. Generate JWT token
    │  4. { "token": "eyJhb..." }       │
    │ ←─────────────────────────────────│
    │                                   │
    │  5. GET /api/orders               │
    │     Header: Authorization:        │
    │     Bearer eyJhb...               │
    │ ─────────────────────────────────→│
    │                                   │  6. Extract token from header
    │                                   │  7. Verify signature
    │                                   │  8. Check expiration
    │                                   │  9. Extract user info
    │                                   │  10. Process request
    │  11. { orders: [...] }            │
    │ ←─────────────────────────────────│
```

---

## 9.5 Spring Security — Implementation Overview

### Step 1: Dependencies (pom.xml)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- For JWT -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

### Step 2: JWT Utility Class

```java
@Component
public class JwtUtil {

    @Value("${app.jwt.secret}")
    private String secretKey;

    @Value("${app.jwt.expiration-ms}")
    private long expirationMs;

    // Generate a token
    public String generateToken(String username, String role) {
        return Jwts.builder()
                .subject(username)
                .claim("role", role)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + expirationMs))
                .signWith(getSigningKey())
                .compact();
    }

    // Extract username from token
    public String extractUsername(String token) {
        return extractClaims(token).getSubject();
    }

    // Validate token
    public boolean isTokenValid(String token) {
        try {
            extractClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }

    private Claims extractClaims(String token) {
        return Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secretKey);
        return Keys.hmacShaKeyFor(keyBytes);
    }
}
```

### Step 3: JWT Authentication Filter

This filter runs **before every request** to check for a valid token:

```java
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtUtil jwtUtil, UserDetailsService userDetailsService) {
        this.jwtUtil = jwtUtil;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {

        // 1. Extract token from "Authorization: Bearer <token>" header
        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response);  // No token → continue (might be a public endpoint)
            return;
        }

        String token = authHeader.substring(7);  // Remove "Bearer " prefix

        // 2. Validate token
        if (jwtUtil.isTokenValid(token)) {
            String username = jwtUtil.extractUsername(token);

            // 3. Load user details
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            // 4. Set authentication in Spring Security context
            UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(
                            userDetails, null, userDetails.getAuthorities());

            SecurityContextHolder.getContext().setAuthentication(authToken);
        }

        filterChain.doFilter(request, response);
    }
}
```

### Step 4: Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // Enables @PreAuthorize, @Secured
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter) {
        this.jwtAuthFilter = jwtAuthFilter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF (not needed for stateless JWT APIs)
            .csrf(csrf -> csrf.disable())

            // Set session management to stateless (no server-side sessions)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Define authorization rules
            .authorizeHttpRequests(auth -> auth
                // Public endpoints — no auth required
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()

                // Role-based access
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/manager/**").hasAnyRole("ADMIN", "MANAGER")

                // Everything else requires authentication
                .anyRequest().authenticated()
            )

            // Add JWT filter before Spring's default auth filter
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();  // NEVER store plain-text passwords
    }
}
```

### Step 5: Auth Controller (Login & Register)

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtUtil jwtUtil;
    private final UserService userService;

    public AuthController(AuthenticationManager authenticationManager,
                          JwtUtil jwtUtil, UserService userService) {
        this.authenticationManager = authenticationManager;
        this.jwtUtil = jwtUtil;
        this.userService = userService;
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
        // Authenticate using Spring Security
        authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                        request.getUsername(), request.getPassword()));

        // Generate JWT
        String token = jwtUtil.generateToken(request.getUsername(), "USER");

        return ResponseEntity.ok(new AuthResponse(token));
    }

    @PostMapping("/register")
    public ResponseEntity<String> register(@Valid @RequestBody RegisterRequest request) {
        userService.registerUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body("User registered successfully");
    }
}
```

---

## 9.6 Method-Level Security

Beyond URL-based rules, you can secure individual methods:

```java
@Service
public class AdminService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long userId) {
        // Only users with ADMIN role can call this
    }

    @PreAuthorize("hasAnyRole('ADMIN', 'MANAGER')")
    public List<Report> getReports() {
        // ADMIN or MANAGER can call this
    }

    @PreAuthorize("#userId == authentication.principal.id")
    public UserProfile getProfile(Long userId) {
        // Users can only view their OWN profile
    }
}
```

---

## 9.7 Password Storage

**NEVER store plain-text passwords.**

```java
// ❌ NEVER DO THIS
user.setPassword("mysecret123");

// ✅ ALWAYS HASH
@Service
public class UserService {

    private final PasswordEncoder passwordEncoder;

    public void registerUser(RegisterRequest request) {
        User user = new User();
        user.setUsername(request.getUsername());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        //                ^^^^^^^^^^^^^^^^^^^^^^^^
        //                BCrypt hashes it: "$2a$10$N9qo8uLOickgx2ZMRZoMye..."
        userRepository.save(user);
    }
}
```

**BCrypt** is the standard:
- Produces a different hash every time (includes random salt)
- Intentionally slow (prevents brute force)
- One-way — can't reverse the hash to get the password

---

## 9.8 The Spring Security Filter Chain

Every HTTP request passes through a **chain of filters** before reaching your controller:

```
HTTP Request
    │
    ▼
┌───────────────────────────────┐
│ 1. CORS Filter                │  Handle cross-origin requests
├───────────────────────────────┤
│ 2. CSRF Filter                │  Cross-site request forgery protection (disabled for JWT APIs)
├───────────────────────────────┤
│ 3. JWT Authentication Filter  │  YOUR custom filter — extracts and validates JWT
├───────────────────────────────┤
│ 4. Authorization Filter       │  Checks roles/permissions
├───────────────────────────────┤
│ 5. Exception Translation      │  Converts security exceptions to HTTP responses
└───────────────────────────────┘
    │
    ▼
Your Controller
```

---

## 9.9 CORS (Cross-Origin Resource Sharing)

### Layman Analogy

```
Your frontend lives at:  https://myapp.com       (Port 3000)
Your backend lives at:   https://api.myapp.com   (Port 8080)

The browser says: "These are DIFFERENT origins! I won't let the frontend
talk to the backend unless the backend explicitly says it's OK."

That permission is CORS.
```

### Configuration

```java
@Configuration
public class CorsConfig {

    @Bean
    public WebMvcConfigurer corsConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                registry.addMapping("/api/**")
                        .allowedOrigins("https://myapp.com", "http://localhost:3000")
                        .allowedMethods("GET", "POST", "PUT", "DELETE", "PATCH")
                        .allowedHeaders("*")
                        .allowCredentials(true)
                        .maxAge(3600);  // Cache preflight for 1 hour
            }
        };
    }
}
```

---

## 9.10 Common Interview Questions & Answers

**Q: What is the difference between authentication and authorization?**
> **Authentication** = verifying identity ("Who are you?" → 401 if fails). **Authorization** = verifying permission ("What can you do?" → 403 if fails). Authentication always comes first.

**Q: Why use JWT over session-based auth for REST APIs?**
> JWT is **stateless** — the server doesn't store anything. This makes it easy to scale (any server can verify the token). Sessions require shared state across servers (sticky sessions or a session store like Redis).

**Q: What is the structure of a JWT?**
> Three parts: **Header** (algorithm), **Payload** (claims — username, role, expiry), **Signature** (ensures token hasn't been tampered with). All Base64-encoded, separated by dots.

**Q: Where should JWTs be stored on the client?**
> **HttpOnly cookies** (best — prevents XSS attacks) or **localStorage** (simpler but vulnerable to XSS). Never in sessionStorage or plain cookies.

**Q: How do you handle token expiration?**
> Use short-lived **access tokens** (15-30 min) and long-lived **refresh tokens** (7 days). When the access token expires, the client uses the refresh token to get a new one without re-logging in.

**Q: Why disable CSRF for JWT-based APIs?**
> CSRF attacks exploit browser cookies. JWT tokens are sent in the `Authorization` header, not cookies, so CSRF doesn't apply. If you store JWT in cookies, you DO need CSRF protection.

**Q: What is BCrypt and why use it?**
> BCrypt is a password hashing algorithm that's intentionally **slow** (configurable rounds), includes a **random salt**, and produces a **different hash** each time. This prevents rainbow table attacks and brute force.

---

## 9.11 Quick Revision Table

| Concept | One-Liner |
|---------|-----------|
| Authentication | Verifying WHO you are (401 on failure) |
| Authorization | Verifying WHAT you can do (403 on failure) |
| JWT | A signed token containing user info — stateless auth |
| JWT Structure | Header.Payload.Signature (Base64 encoded, dot separated) |
| `SecurityFilterChain` | Configures URL-based access rules |
| `@PreAuthorize` | Method-level access control |
| BCrypt | Password hashing — slow, salted, one-way |
| CORS | Browser policy — server must explicitly allow cross-origin requests |
| Access Token | Short-lived JWT for API calls (15-30 min) |
| Refresh Token | Long-lived token to get new access tokens (7+ days) |
| `OncePerRequestFilter` | Custom filter that runs once per request (used for JWT validation) |

---

### Interview Tip 🗣️

> "Spring Security handles both authentication (who are you) and authorization (what can you do). For REST APIs, I use **JWT** — it's stateless, scalable, and doesn't require server-side sessions. The JWT filter validates the token on every request, the SecurityFilterChain defines URL access rules, and `@PreAuthorize` secures individual methods. Passwords are always BCrypt-hashed."

---

[⬅️ Back to Curriculum](./README.md) | [⬅️ Previous: REST Best Practices](./08-rest-api-best-practices.md) | [Next: Microservices vs Monolith ➡️](./10-microservices-vs-monolith.md)
