---
name: backend-dev-guidelines-go-fiber
description: Comprehensive backend development guide for Go/Fiber HTTP APIs. Use when creating routes, handlers, services, middleware, or working with Fiber APIs, database access with database/sql, validation patterns, error handling, testing strategies, and performance optimization. Covers layered architecture (routes → handlers → services → database), middleware patterns, security best practices, and Go idioms.
---

# Go Backend Development Guidelines (Fiber Framework)

## Purpose

Establish consistency and best practices for Go HTTP APIs using Fiber framework, following Go idioms and industry standards for security, performance, and maintainability.

## When to Use This Skill

Automatically activates when working on:
- Creating or modifying routes, endpoints, HTTP handlers
- Building services with business logic
- Implementing middleware (auth, rate limiting, metrics, logging)
- Database operations with database/sql or ORMs
- Input validation and error handling
- Security hardening (SSRF, injection prevention, auth)
- Backend testing and performance optimization

---

## Quick Start

### New Backend Feature Checklist

- [ ] **Route**: Clean Fiber route registration
- [ ] **Handler**: Request handler with proper error handling
- [ ] **Service**: Business logic layer
- [ ] **Database**: Database access with proper transaction handling
- [ ] **Validation**: Input validation with security checks
- [ ] **Error Handling**: Structured errors with proper logging
- [ ] **Tests**: Unit + integration tests
- [ ] **Security**: SSRF protection, input sanitization, auth checks

### New API Service Checklist

- [ ] Directory structure (see [architecture-overview.md](architecture-overview.md))
- [ ] main.go with Fiber app setup
- [ ] Middleware stack (logger, security headers, CORS)
- [ ] Error handling patterns
- [ ] Database initialization
- [ ] Configuration from environment
- [ ] Testing framework setup
- [ ] Security hardening (rate limiting, API keys, SSRF)

---

## Architecture Overview

### Layered Architecture

```
HTTP Request
    ↓
Routes (routing only)
    ↓
Handlers (request/response handling)
    ↓
Services (business logic)
    ↓
Database (database/sql or ORM)
```

**Key Principle:** Each layer has ONE responsibility. Handlers handle HTTP, services contain business logic, database layer manages data access.

See [architecture-overview.md](architecture-overview.md) for complete details.

---

## Directory Structure

```
api/
├── main.go              # Application entry point
├── routes/              # Route registration & handlers
│   ├── health.go
│   ├── users.go
│   └── products.go
├── services/            # Business logic layer
│   ├── user_service.go
│   └── product_service.go
├── middleware/          # HTTP middleware
│   ├── auth.go
│   ├── rate_limit.go
│   └── logger.go
├── db/                  # Database layer
│   ├── database.go
│   ├── users.go
│   └── products.go
├── docs/                # Swagger docs (generated)
└── go.mod               # Dependencies
```

**Naming Conventions:**
- Packages: lowercase, no underscores (`middleware`, `routes`)
- Files: snake_case (`user_service.go`, `rate_limit.go`)
- Handlers: verb + descriptive name (`handleCreateUser`, `handleGetProduct`, `HealthHandler`)
- Services: descriptive + purpose (`CreateUser`, `GetProductByID`, `DeleteProduct`)

---

## Core Principles (7 Key Rules)

### 1. Routes Only Route, Handlers Handle

```go
// ❌ NEVER: Business logic in route registration
app.Post("/users", func(c *fiber.Ctx) error {
    // 200 lines of user creation logic
})

// ✅ ALWAYS: Delegate to handler function
app.Post("/users", handleCreateUser)

// Or for complex initialization:
app.Post("/users", routes.CreateUserHandler())
```

### 2. Handlers Return Errors, Never Panic

```go
// ✅ ALWAYS: Return errors for Fiber to handle
func handleCreateUser(c *fiber.Ctx) error {
    user, err := createUserFromRequest(c)
    if err != nil {
        log.Printf("Error creating user: %v", err)
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to create user",
        })
    }
    return c.Status(fiber.StatusCreated).JSON(user)
}
```

### 3. All Errors Logged with Context

```go
// ✅ ALWAYS: Log errors with contextual information
log.Printf("[ERROR] Failed to create user - IP: %s, Path: %s, Error: %v",
    c.IP(), c.Path(), err)

// For security events:
log.Printf("[SECURITY] SSRF attempt blocked - IP: %s, URL: %s", c.IP(), url)
```

### 4. Use Environment Variables for Config

```go
// ❌ NEVER: Hardcoded values
const timeout = 5000

// ✅ ALWAYS: Load from environment
timeout := os.Getenv("TIMEOUT_MS")
if timeout == "" {
    timeout = "5000" // Sensible default
}
```

### 5. Validate ALL Input

```go
// ✅ ALWAYS: Validate query parameters, form data, headers
quality, err := strconv.Atoi(c.Query("quality"))
if err != nil || quality < 1 || quality > 100 {
    return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
        "error": "Invalid quality parameter. Must be between 1 and 100.",
    })
}
```

### 6. Use Service Layer for Business Logic

```go
// ✅ Handlers delegate to services
func handleCreateUser(c *fiber.Ctx) error {
    userData := extractUserData(c) // Extract from request
    user, err := services.CreateUser(userData)
    if err != nil {
        return handleError(c, err)
    }
    return c.Status(fiber.StatusCreated).JSON(user)
}
```

### 7. Comprehensive Testing Required

```go
func TestCreateUser(t *testing.T) {
    userData := UserData{Email: "test@example.com", Name: "Test User"}
    user, err := services.CreateUser(userData)
    if err != nil {
        t.Fatalf("Expected no error, got: %v", err)
    }
    if user.ID == 0 {
        t.Errorf("Expected user ID to be set")
    }
}
```

---

## Common Imports

```go
// HTTP Framework
import (
    "github.com/gofiber/fiber/v2"
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/compress"
)

// Standard library
import (
    "context"
    "database/sql"
    "fmt"
    "log"
    "os"
    "strconv"
    "strings"
    "time"
)

// Database
import (
    _ "github.com/mattn/go-sqlite3"  // SQLite driver
    _ "github.com/lib/pq"             // PostgreSQL driver
)

// Testing
import (
    "testing"
    "net/http/httptest"
)
```

---

## Quick Reference

### HTTP Status Codes

| Code | Use Case |
|------|----------|
| 200 | Success |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request (validation failed) |
| 401 | Unauthorized (missing/invalid auth) |
| 403 | Forbidden (valid auth, insufficient permissions) |
| 404 | Not Found |
| 413 | Payload Too Large |
| 429 | Too Many Requests |
| 500 | Internal Server Error |

### Fiber Response Patterns

```go
// JSON response
return c.JSON(fiber.Map{"status": "ok"})

// With status code
return c.Status(fiber.StatusCreated).JSON(result)

// Send file/binary
c.Type("image/jpeg")
return c.Send(imageBytes)

// Redirect
return c.Redirect("/new-path", fiber.StatusMovedPermanently)
```

---

## Anti-Patterns to Avoid

❌ Business logic in handlers
❌ Hardcoded configuration values
❌ Missing error handling or logging
❌ No input validation
❌ Direct panic in handlers (return errors instead)
❌ Missing security checks (SSRF, injection, auth)
❌ No tests

---

## Navigation Guide

| Need to... | Read this |
|------------|-----------|
| Understand architecture | [architecture-overview.md](architecture-overview.md) |
| Create routes/handlers | [routing-and-handlers.md](routing-and-handlers.md) |
| Organize business logic | [services-and-patterns.md](services-and-patterns.md) |
| Validate input | [validation-patterns.md](validation-patterns.md) |
| Handle errors | [error-handling.md](error-handling.md) |
| Create middleware | [middleware-guide.md](middleware-guide.md) |
| Database access | [database-patterns.md](database-patterns.md) |
| Manage config | [configuration.md](configuration.md) |
| Security hardening | [security-guide.md](security-guide.md) |
| Write tests | [testing-guide.md](testing-guide.md) |
| Performance optimization | [performance-guide.md](performance-guide.md) |

---

## Resource Files

### [architecture-overview.md](architecture-overview.md)
Layered architecture, request lifecycle, separation of concerns, Go project structure

### [routing-and-handlers.md](routing-and-handlers.md)
Fiber routing, handler patterns, error handling, request/response examples

### [services-and-patterns.md](services-and-patterns.md)
Service layer patterns, business logic organization, Go idioms

### [validation-patterns.md](validation-patterns.md)
Input validation, security checks, parameter parsing

### [error-handling.md](error-handling.md)
Error patterns, logging, structured errors, error responses

### [middleware-guide.md](middleware-guide.md)
Auth, rate limiting, logging, metrics, security headers

### [database-patterns.md](database-patterns.md)
database/sql, transactions, connection pooling, migrations

### [configuration.md](configuration.md)
Environment variables, configuration management, secrets

### [security-guide.md](security-guide.md)
SSRF protection, API keys, input sanitization, security headers

### [testing-guide.md](testing-guide.md)
Unit/integration tests, mocking, test coverage, httptest

### [performance-guide.md](performance-guide.md)
Concurrency, memory management, profiling, optimization

---

**Skill Status**: COMPLETE ✅
**Line Count**: < 500 ✅
**Progressive Disclosure**: 11 resource files ✅
