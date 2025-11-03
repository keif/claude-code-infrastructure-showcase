# Architecture Overview

## Go API Architecture Principles

### Layered Architecture Pattern

```
┌─────────────────────────────────────┐
│   HTTP Request (Fiber Context)     │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│         Routes Layer                │
│  • Route registration               │
│  • Path matching                    │
│  • Method routing (GET/POST/etc)    │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│        Middleware Layer             │
│  • Authentication (API keys)        │
│  • Rate limiting                    │
│  • Logging                          │
│  • Metrics collection               │
│  • CORS                             │
│  • Security headers                 │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│        Handlers Layer               │
│  • Request parsing                  │
│  • Input validation                 │
│  • Response formatting              │
│  • Error handling                   │
│  • Delegate to services             │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│        Services Layer               │
│  • Business logic                   │
│  • Data processing                  │
│  • External API calls               │
│  • Complex calculations             │
│  • Orchestration                    │
└──────────────┬──────────────────────┘
               │
┌──────────────▼──────────────────────┐
│        Database Layer               │
│  • SQL queries                      │
│  • Transactions                     │
│  • Connection pooling               │
│  • Data access                      │
└───────────────────────────────────── ┘
```

---

## Standard Go API Project Structure

```
api/
├── main.go                 # Application entry point
│   • Fiber app initialization
│   • Middleware setup
│   • Route registration
│   • Server startup
│
├── routes/                 # Route handlers
│   ├── health.go          # Health check endpoint
│   ├── users.go        # User management routes
│   ├── admin.go           # Admin routes
│   └── api_keys.go        # API key management
│
├── services/              # Business logic
│   ├── user_service.go   # User management logic
│   └── auth_service.go    # Authentication logic
│
├── middleware/            # HTTP middleware
│   ├── api_key.go        # API key authentication
│   ├── rate_limit.go     # Rate limiting
│   ├── metrics.go        # Metrics collection
│   └── logger.go         # Request logging
│
├── db/                    # Database layer
│   ├── database.go       # DB initialization
│   ├── api_keys.go       # API key queries
│   └── metrics.go        # Metrics queries
│
├── docs/                  # Auto-generated Swagger docs
│   └── docs.go
│
├── data/                  # Local data (gitignored)
│   └── *.db
│
├── go.mod                 # Dependencies
├── go.sum                 # Dependency checksums
└── README.md
```

---

## Request Lifecycle

### 1. Request Arrives

```go
// Incoming HTTP request to POST /users
```

### 2. Middleware Chain Executes

```go
// main.go
app.Use(logger.New())                    // Log request
app.Use(middleware.RequireAPIKey())      // Validate API key
app.Use(middleware.NewRateLimiter())     // Check rate limit
app.Use(middleware.NewMetricsCollector()) // Collect metrics
```

### 3. Route Handler Matches

```go
// routes/users.go
app.Post("/users", handleCreateUser)
```

### 4. Handler Processes Request

```go
func handleCreateUser(c *fiber.Ctx) error {
    // 1. Parse request parameters
    options := parseOptimizeOptions(c)

    // 2. Validate input
    if err := validateOptions(options); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": err.Error(),
        })
    }

    // 3. Call service layer
    result, err := services.CreateUser(data, options)
    if err != nil {
        log.Printf("Error: %v", err)
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to process request",
        })
    }

    // 4. Return response
    return c.JSON(result)
}
```

### 5. Service Executes Business Logic

```go
// services/user_service.go
func CreateUser(data []byte, opts UserOptions) (*Result, error) {
    // Business logic here
    // - Process data
    // - Apply business rules
    // - Generate result
    return &Result{...}, nil
}
```

### 6. Database Access (if needed)

```go
// db/metrics.go
func RecordMetric(timestamp time.Time, endpoint string, ...) error {
    _, err := db.DB.Exec(`
        INSERT INTO metrics (timestamp, endpoint, ...)
        VALUES (?, ?, ...)
    `, timestamp, endpoint, ...)
    return err
}
```

### 7. Response Returns to Client

```go
// Fiber handles serialization and sends response
```

---

## Separation of Concerns

### DO: Clear Separation

```go
// routes/user.go - Handles HTTP only
func handleGetUser(c *fiber.Ctx) error {
    id := c.Params("id")
    user, err := services.GetUser(id)
    if err != nil {
        return handleError(c, err)
    }
    return c.JSON(user)
}

// services/user_service.go - Business logic only
func GetUser(id string) (*User, error) {
    return db.FindUserByID(id)
}

// db/users.go - Data access only
func FindUserByID(id string) (*User, error) {
    var user User
    err := DB.QueryRow("SELECT * FROM users WHERE id = ?", id).Scan(&user)
    return &user, err
}
```

### DON'T: Mixed Concerns

```go
// ❌ Handler with business logic AND database access
func handleGetUser(c *fiber.Ctx) error {
    id := c.Params("id")

    // Business logic in handler - BAD
    if !isValidID(id) {
        return c.Status(400).JSON(fiber.Map{"error": "invalid"})
    }

    // Database access in handler - BAD
    var user User
    err := DB.QueryRow("SELECT * FROM users WHERE id = ?", id).Scan(&user)
    if err != nil {
        return c.Status(500).JSON(fiber.Map{"error": "db error"})
    }

    return c.JSON(user)
}
```

---

## Package Organization

### Package Naming

- **Lowercase**: All package names lowercase, no underscores
- **Singular**: Prefer singular form (`db`, `service`, `middleware`)
- **Short**: Keep package names concise (`routes` not `route_handlers`)

```go
// ✅ Good
package routes
package middleware
package db

// ❌ Bad
package RouteHandlers
package middle_ware
package database_access_layer
```

### Internal Packages

For private packages not meant to be imported by others:

```
api/
├── internal/
│   ├── validation/    # Internal validation helpers
│   └── util/         # Internal utilities
```

---

## Configuration Management

### Environment-Based Config

```go
// main.go
port := os.Getenv("PORT")
if port == "" {
    port = "8080" // Default
}

dbPath := os.Getenv("DB_PATH")
if dbPath == "" {
    dbPath = "./data/api.db"
}
```

### Config Struct Pattern

```go
type Config struct {
    Port          string
    DBPath        string
    AllowedOrigins []string
    RateLimit     int
}

func LoadConfig() *Config {
    return &Config{
        Port:   getEnv("PORT", "8080"),
        DBPath: getEnv("DB_PATH", "./data/api.db"),
        AllowedOrigins: strings.Split(getEnv("CORS_ORIGINS", "http://localhost:3000"), ","),
        RateLimit: getEnvInt("RATE_LIMIT", 100),
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

---

## Error Handling Philosophy

### Errors Are Values

In Go, errors are values that must be explicitly handled:

```go
// ✅ Always check errors
result, err := service.Process(data)
if err != nil {
    log.Printf("Error processing: %v", err)
    return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
        "error": "Processing failed",
    })
}
```

### Avoid Panics in HTTP Handlers

```go
// ❌ Never panic in handlers
func handleRequest(c *fiber.Ctx) error {
    panic("This will crash the server!")
}

// ✅ Return errors instead
func handleRequest(c *fiber.Ctx) error {
    if criticalError {
        log.Printf("Critical error occurred")
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Internal error",
        })
    }
    return c.JSON(result)
}
```

---

## Concurrency Patterns

### Goroutines for Async Work

```go
// Parallel processing with worker pool
func processBatch(files []File) []Result {
    numWorkers := 4
    jobs := make(chan File, len(files))
    results := make(chan Result, len(files))

    // Start workers
    for w := 0; w < numWorkers; w++ {
        go func() {
            for file := range jobs {
                result := processFile(file)
                results <- result
            }
        }()
    }

    // Send jobs
    for _, file := range files {
        jobs <- file
    }
    close(jobs)

    // Collect results
    var allResults []Result
    for i := 0; i < len(files); i++ {
        allResults = append(allResults, <-results)
    }
    close(results)

    return allResults
}
```

### Semaphores for Resource Limiting

```go
// Limit concurrent large operations
var largeSemaphore = make(chan struct{}, 2) // Max 2 concurrent

func processLarge(data []byte) error {
    largeSemaphore <- struct{}{}        // Acquire
    defer func() { <-largeSemaphore }() // Release

    // Process large data
    return nil
}
```

---

## Go Idioms to Follow

1. **Accept interfaces, return structs**
2. **Handle errors explicitly**
3. **Use defer for cleanup**
4. **Keep functions small and focused**
5. **Prefer composition over inheritance**
6. **Use channels for communication**
7. **Document exported functions**
8. **Run gofmt/goimports**

---

## Related Resources

- [routing-and-handlers.md](routing-and-handlers.md) - HTTP routing patterns
- [services-and-patterns.md](services-and-patterns.md) - Service layer design
- [middleware-guide.md](middleware-guide.md) - Middleware patterns
- [database-patterns.md](database-patterns.md) - Database access
- [error-handling.md](error-handling.md) - Error patterns
