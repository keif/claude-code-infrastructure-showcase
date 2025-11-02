# Middleware Guide

## Fiber Middleware Patterns

### Basic Middleware

```go
func MyMiddleware() fiber.Handler {
    return func(c *fiber.Ctx) error {
        // Before request
        log.Printf("Request: %s %s", c.Method(), c.Path())

        // Continue to next handler
        err := c.Next()

        // After request
        log.Printf("Response: %d", c.Response().StatusCode())

        return err
    }
}
```

### Middleware with Configuration

```go
type RateLimitConfig struct {
    RequestsPerSecond int
    BurstSize         int
}

func NewRateLimiter(config RateLimitConfig) fiber.Handler {
    limiter := rate.NewLimiter(
        rate.Limit(config.RequestsPerSecond),
        config.BurstSize,
    )

    return func(c *fiber.Ctx) error {
        if !limiter.Allow() {
            return c.Status(429).JSON(fiber.Map{
                "error": "Rate limit exceeded",
            })
        }
        return c.Next()
    }
}
```

### Built-in Middleware

```go
import (
    "github.com/gofiber/fiber/v2/middleware/logger"
    "github.com/gofiber/fiber/v2/middleware/compress"
    "github.com/gofiber/fiber/v2/middleware/cors"
)

app.Use(logger.New())
app.Use(compress.New())
app.Use(cors.New(cors.Config{
    AllowOrigins: "https://example.com",
}))
```
