# Error Handling

## Go Error Patterns

### Basic Error Handling

```go
result, err := service.Process(data)
if err != nil {
    log.Printf("Error processing: %v", err)
    return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
        "error": "Processing failed",
    })
}
```

### Custom Error Types

```go
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("%s: %s", e.Field, e.Message)
}
```

### Error Wrapping

```go
if err != nil {
    return fmt.Errorf("failed to process image: %w", err)
}
```

### Logging Errors with Context

```go
log.Printf("[ERROR] Operation failed - IP: %s, Path: %s, Error: %v",
    c.IP(), c.Path(), err)
```

## Error Response Patterns

### Development vs Production

```go
func errorResponse(c *fiber.Ctx, status int, message string, err error) error {
    response := fiber.Map{
        "error": message,
    }

    // Include details in development only
    if os.Getenv("GO_ENV") != "production" && err != nil {
        response["details"] = err.Error()
    }

    return c.Status(status).JSON(response)
}
```
