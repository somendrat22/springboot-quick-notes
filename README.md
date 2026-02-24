# Backend Engineering Interview Prep Curriculum

> A step-by-step guide with **layman examples** to help you ace your backend engineering interview.

---

## Who Is This For?

Anyone preparing for a **backend engineer interview** — especially those working with **Java / Spring Boot**. Every concept is explained with a real-world analogy first, then with code.

---

## Curriculum Structure

This curriculum is organized into **11 modules**. Study them **in order** — each builds on the previous one.

### Part 1: Core Concepts

| # | Module | File | Time Estimate |
|---|--------|------|---------------|
| 1 | Client-Server Architecture | [01-client-server-architecture.md](./01-client-server-architecture.md) | 20 min |
| 2 | Dependency Injection & IoC Container | [02-dependency-injection-ioc.md](./02-dependency-injection-ioc.md) | 30 min |
| 3 | Controller → Service → Repository | [03-controller-service-repository.md](./03-controller-service-repository.md) | 40 min |
| 4 | JPA Repository | [04-jpa-repository.md](./04-jpa-repository.md) | 35 min |

### Part 2: Production Essentials

| # | Module | File | Time Estimate |
|---|--------|------|---------------|
| 5 | Exception Handling | [05-exception-handling.md](./05-exception-handling.md) | 25 min |
| 6 | DTO & Mapping | [06-dto-and-mapping.md](./06-dto-and-mapping.md) | 25 min |
| 7 | Spring Boot Annotations Cheat Sheet | [07-spring-boot-annotations.md](./07-spring-boot-annotations.md) | 20 min |
| 8 | REST API Best Practices | [08-rest-api-best-practices.md](./08-rest-api-best-practices.md) | 25 min |
| 9 | Spring Security Basics | [09-spring-security-basics.md](./09-spring-security-basics.md) | 30 min |
| 10 | Microservices vs Monolith | [10-microservices-vs-monolith.md](./10-microservices-vs-monolith.md) | 30 min |

### Part 3: System Design & Real-World Projects

| # | Module | File | Time Estimate |
|---|--------|------|---------------|
| 11 | Project Scenarios — DB Design & Endpoints | [11-project-scenarios-db-design.md](./11-project-scenarios-db-design.md) | 60 min |

**Total estimated study time: ~5.5 hours**

---

## How to Use This Guide

1. **Read the layman example first** — understand the concept in plain English.
2. **Then look at the code** — map the analogy to real Spring Boot code.
3. **Practice the "Interview Tip"** at the end of each section — these are one-liner answers you can use in the actual interview.
4. **Use the Quick Revision table** in each file for last-minute review.

---

## Quick Revision — All Topics in One Glance

| Topic | One-Liner |
|-------|-----------|
| **Client-Server** | Client asks, server answers — they talk over HTTP. |
| **Dependency Injection** | Don't create your own dependencies — let the framework hand them to you. |
| **IoC Container** | Spring's ApplicationContext — the matchmaker that creates and wires all beans. |
| **Controller** | The front door — receives HTTP requests and returns HTTP responses. |
| **Service** | The brain — contains business rules and logic. |
| **Repository** | The bridge to the database — handles all data access. |
| **JPA** | A spec for mapping Java objects to DB tables; Hibernate implements it. |
| **JpaRepository** | Extend it, name your methods correctly, and Spring writes the SQL for you. |
| **Exception Handling** | `@ControllerAdvice` catches all exceptions globally — controllers stay clean. |
| **DTO** | A filtered data carrier — never expose your Entity directly to the client. |
| **Annotations** | Metadata tags that tell Spring what to create, wire, route, and manage. |
| **REST Best Practices** | Plural nouns, proper status codes, pagination, versioning, consistent errors. |
| **Authentication** | Proving WHO you are (JWT token, 401 on failure). |
| **Authorization** | Checking WHAT you can do (roles/permissions, 403 on failure). |
| **Monolith** | One codebase, one deployment — simple but doesn't scale well. |
| **Microservices** | Many small services with own DBs — scalable but complex. |
| **Circuit Breaker** | Stop calling a failing service, return fallback instead. |
| **DB Design** | Start with entities & relationships, snapshot prices, soft delete, index FKs. |
| **Pessimistic Locking** | Lock DB rows during money operations to prevent race conditions. |
| **Idempotency** | Use a unique reference ID so retries don't create duplicate transactions. |

---

**Good luck for the interview! You've got this! 🚀**
