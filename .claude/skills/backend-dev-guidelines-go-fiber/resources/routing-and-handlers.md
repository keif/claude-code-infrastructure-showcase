# Routing and Handlers

## Fiber Routing Patterns

### Basic Route Registration

```go
// main.go
app := fiber.New()

// Simple routes
app.Get("/health", handleHealth)
app.Post("/users", handleCreateUser)
app.Delete("/api/keys/:id", handleDeleteKey)

// Route groups
api := app.Group("/api")
api.Get("/keys", handleListKeys)
api.Post("/keys", handleCreateKey)
api.Delete("/keys/:id", handleDeleteKey)

// With middleware
admin := app.Group("/admin", middleware.RequireAdmin())
admin.Get("/metrics", handleMetrics)
admin.Post("/config", handleUpdateConfig)
```

### Route Organization

```go
// routes/users.go
package routes

func RegisterUserRoutes(app *fiber.App) {
    app.Post("/users", handleCreateUser)
    app.Post("/batch-users", handleBatchCreateUsers)
}

// routes/api_keys.go
func RegisterAPIKeyRoutes(app *fiber.App) {
    app.Post("/api/keys", handleCreateAPIKey)
    app.Get("/api/keys", handleListAPIKeys)
    app.Delete("/api/keys/:id", handleRevokeAPIKey)
}

// main.go
routes.RegisterUserRoutes(app)
routes.RegisterAPIKeyRoutes(app)
```

---

## Handler Patterns

### Standard Handler Signature

```go
func handleRequest(c *fiber.Ctx) error {
    // Handler implementation
    return c.JSON(fiber.Map{"status": "ok"})
}
```

### Complete Handler Example

```go
func handleCreateUser(c *fiber.Ctx) error {
    // 1. Parse and validate input
    quality, err := parseQuality(c)
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": err.Error(),
        })
    }

    // 2. Get request data
    file, err := c.FormFile("image")
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Missing image file",
        })
    }

    // 3. Read file
    f, err := file.Open()
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Failed to open file",
        })
    }
    defer f.Close()

    imgData, err := io.ReadAll(io.LimitReader(f, maxFileSize))
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Failed to read file",
        })
    }

    // 4. Call service layer
    result, err := services.CreateUser(imgData, services.UserOptions{
        Quality: quality,
    })
    if err != nil {
        log.Printf("Error processing request: %v", err)
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to process request",
        })
    }

    // 5. Return response
    return c.JSON(result)
}
```

---

## Request Parsing

### Query Parameters

```go
// GET /search?query=test&limit=10&offset=0

func handleSearch(c *fiber.Ctx) error {
    // String parameters
    query := c.Query("query")
    if query == "" {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Missing query parameter",
        })
    }

    // Integer parameters
    limitStr := c.Query("limit", "10") // With default
    limit, err := strconv.Atoi(limitStr)
    if err != nil || limit < 1 || limit > 100 {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid limit parameter",
        })
    }

    // Boolean parameters
    includeDeleted := c.QueryBool("includeDeleted", false)

    results := services.Search(query, limit, includeDeleted)
    return c.JSON(results)
}
```

### Path Parameters

```go
// DELETE /api/keys/:id

func handleDeleteKey(c *fiber.Ctx) error {
    id := c.Params("id")
    if id == "" {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Missing ID parameter",
        })
    }

    err := db.DeleteAPIKey(id)
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to delete key",
        })
    }

    return c.SendStatus(fiber.StatusNoContent)
}
```

### Form Data

```go
// POST with multipart/form-data

func handleUpload(c *fiber.Ctx) error {
    // Single file
    file, err := c.FormFile("image")
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Missing image file",
        })
    }

    // Form values
    title := c.FormValue("title")
    description := c.FormValue("description")

    // Multiple files
    form, err := c.MultipartForm()
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid multipart form",
        })
    }
    files := form.File["images"]

    // Process files...
    return c.JSON(fiber.Map{"uploaded": len(files)})
}
```

### JSON Body

```go
type CreateUserRequest struct {
    Email    string `json:"email"`
    Name     string `json:"name"`
    Password string `json:"password"`
}

func handleCreateUser(c *fiber.Ctx) error {
    var req CreateUserRequest

    // Parse JSON body
    if err := c.BodyParser(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid JSON",
        })
    }

    // Validate
    if req.Email == "" || req.Password == "" {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Email and password required",
        })
    }

    // Create user
    user, err := services.CreateUser(req.Email, req.Name, req.Password)
    if err != nil {
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to create user",
        })
    }

    return c.Status(fiber.StatusCreated).JSON(user)
}
```

---

## Response Patterns

### JSON Responses

```go
// Success response
return c.JSON(fiber.Map{
    "status": "success",
    "data":   result,
})

// With status code
return c.Status(fiber.StatusCreated).JSON(fiber.Map{
    "id":   user.ID,
    "name": user.Name,
})

// Error response
return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
    "error": "Invalid input",
})
```

### File Responses

```go
// Return binary data
func handleGetImage(c *fiber.Ctx) error {
    imageData, err := loadImage(c.Params("id"))
    if err != nil {
        return c.Status(fiber.StatusNotFound).JSON(fiber.Map{
            "error": "Image not found",
        })
    }

    // Set content type
    c.Type("image/jpeg")

    // Set content disposition
    c.Set("Content-Disposition", "inline; filename=\"image.jpg\"")

    return c.Send(imageData)
}
```

### Status Only

```go
// 204 No Content
return c.SendStatus(fiber.StatusNoContent)

// 200 OK with custom message
return c.SendString("Operation completed successfully")
```

### Redirects

```go
return c.Redirect("/new-path", fiber.StatusMovedPermanently)
```

---

## Error Handling in Handlers

### Structured Error Responses

```go
func errorResponse(c *fiber.Ctx, status int, message string, err error) error {
    response := fiber.Map{
        "error": message,
    }

    // Include error details in development
    if os.Getenv("GO_ENV") != "production" && err != nil {
        response["details"] = err.Error()
    }

    return c.Status(status).JSON(response)
}

// Usage:
func handleRequest(c *fiber.Ctx) error {
    result, err := service.Process(data)
    if err != nil {
        return errorResponse(c, fiber.StatusInternalServerError, "Processing failed", err)
    }
    return c.JSON(result)
}
```

### Error Handler Middleware

```go
// Global error handler
app.Use(func(c *fiber.Ctx) error {
    err := c.Next()

    if err != nil {
        // Log error
        log.Printf("Error: %v", err)

        // Check if error is a Fiber error
        if e, ok := err.(*fiber.Error); ok {
            return c.Status(e.Code).JSON(fiber.Map{
                "error": e.Message,
            })
        }

        // Default 500 error
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Internal server error",
        })
    }

    return nil
})
```

---

## Request Context

### Getting Client Information

```go
func handleRequest(c *fiber.Ctx) error {
    // Client IP
    ip := c.IP()

    // Request method
    method := c.Method()

    // Request path
    path := c.Path()

    // Headers
    origin := c.Get("Origin")
    userAgent := c.Get("User-Agent")
    authorization := c.Get("Authorization")

    // Protocol
    protocol := c.Protocol() // http or https

    log.Printf("%s %s from %s", method, path, ip)
    return c.SendStatus(fiber.StatusOK)
}
```

### Setting Headers

```go
func handleRequest(c *fiber.Ctx) error {
    // Set header
    c.Set("X-Custom-Header", "value")

    // Set content type
    c.Type("application/json")

    // Set multiple headers
    c.Set("Cache-Control", "no-cache, no-store, must-revalidate")
    c.Set("Pragma", "no-cache")
    c.Set("Expires", "0")

    return c.JSON(data)
}
```

---

## Handler with Closures

### Dependency Injection Pattern

```go
type HandlerDependencies struct {
    DB      *sql.DB
    Config  *Config
    Service *UserService
}

func NewUserHandler(deps *HandlerDependencies) fiber.Handler {
    return func(c *fiber.Ctx) error {
        // Access dependencies via closure
        result, err := deps.Service.ProcessUser(userData)
        if err != nil {
            return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
                "error": "Processing failed",
            })
        }
        return c.JSON(result)
    }
}

// Usage:
deps := &HandlerDependencies{
    DB:      db,
    Config:  config,
    Service: userService,
}
app.Post("/users", NewUserHandler(deps))
```

---

## Streaming Responses

### Server-Sent Events

```go
func handleStream(c *fiber.Ctx) error {
    c.Set("Content-Type", "text/event-stream")
    c.Set("Cache-Control", "no-cache")
    c.Set("Connection", "keep-alive")

    c.Context().SetBodyStreamWriter(func(w *bufio.Writer) {
        for i := 0; i < 10; i++ {
            fmt.Fprintf(w, "data: Message %d\n\n", i)
            w.Flush()
            time.Sleep(time.Second)
        }
    })

    return nil
}
```

---

## Handler Testing

### HTTP Test Pattern

```go
func TestHandleCreateUser(t *testing.T) {
    app := fiber.New()
    app.Post("/users", handleCreateUser)

    // Create test request
    req := httptest.NewRequest("POST", "/users", nil)
    req.Header.Set("Content-Type", "multipart/form-data")

    // Execute request
    resp, err := app.Test(req)
    if err != nil {
        t.Fatal(err)
    }

    // Check response
    if resp.StatusCode != fiber.StatusOK {
        t.Errorf("Expected status 200, got %d", resp.StatusCode)
    }
}
```

---

## Best Practices

1. **Keep handlers thin** - Delegate to services
2. **Validate early** - Check all input at handler level
3. **Return errors** - Never panic in handlers
4. **Log errors** - Include context (IP, path, params)
5. **Use proper status codes** - Be specific (400 vs 422 vs 500)
6. **Handle cleanup** - Use defer for file closing
7. **Set appropriate headers** - Content-Type, Cache-Control
8. **Document with Swagger** - Use comments for auto-docs

---

## Related Resources

- [validation-patterns.md](validation-patterns.md) - Input validation
- [error-handling.md](error-handling.md) - Error patterns
- [middleware-guide.md](middleware-guide.md) - Middleware usage
- [testing-guide.md](testing-guide.md) - Testing handlers
