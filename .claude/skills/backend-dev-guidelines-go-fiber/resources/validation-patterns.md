# Validation Patterns

## Input Validation Philosophy

**VALIDATE EVERYTHING** - Never trust user input. All data from HTTP requests must be validated before use.

---

## Parameter Validation Helpers

### Integer Parameters

```go
func parseIntParam(c *fiber.Ctx, key string, defaultValue, min, max int) (int, error) {
    valueStr := c.Query(key)
    if valueStr == "" {
        return defaultValue, nil
    }

    value, err := strconv.Atoi(valueStr)
    if err != nil {
        return 0, fmt.Errorf("%s must be a number", key)
    }

    if value < min || value > max {
        return 0, fmt.Errorf("%s must be between %d and %d", key, min, max)
    }

    return value, nil
}

// Usage:
quality, err := parseIntParam(c, "quality", 80, 1, 100)
if err != nil {
    return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
        "error": err.Error(),
    })
}
```

### String Parameters

```go
func validateStringParam(value, paramName string, allowed []string) error {
    if value == "" {
        return nil // Optional parameter
    }

    for _, a := range allowed {
        if value == a {
            return nil
        }
    }

    return fmt.Errorf("%s must be one of: %v", paramName, allowed)
}

// Usage:
format := c.Query("format")
err := validateStringParam(format, "format", []string{"jpeg", "png", "webp", "avif"})
if err != nil {
    return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
        "error": err.Error(),
    })
}
```

---

## File Upload Validation

### Complete File Validation

```go
const maxFileSize = 10 * 1024 * 1024 // 10MB

func validateImageFile(c *fiber.Ctx, fieldName string) ([]byte, error) {
    // Get file from form
    file, err := c.FormFile(fieldName)
    if err != nil {
        return nil, fmt.Errorf("missing %s file", fieldName)
    }

    // Check size
    if file.Size > maxFileSize {
        return nil, fmt.Errorf("file exceeds maximum size of 10MB")
    }

    // Validate content type
    validTypes := map[string]bool{
        "image/jpeg": true,
        "image/png":  true,
        "image/webp": true,
        "image/avif": true,
    }

    contentType := file.Header.Get("Content-Type")
    if !validTypes[contentType] {
        return nil, fmt.Errorf("invalid file type: %s", contentType)
    }

    // Read file data
    f, err := file.Open()
    if err != nil {
        return nil, fmt.Errorf("failed to open file")
    }
    defer f.Close()

    // Read with size limit
    data, err := io.ReadAll(io.LimitReader(f, maxFileSize))
    if err != nil {
        return nil, fmt.Errorf("failed to read file")
    }

    // Check if hit size limit
    if len(data) >= int(maxFileSize) {
        return nil, fmt.Errorf("file exceeds maximum size")
    }

    return data, nil
}
```

---

## URL Validation

### Safe URL Parsing

```go
func validateURL(urlStr string) (*url.URL, error) {
    // Parse URL
    parsed, err := url.Parse(urlStr)
    if err != nil {
        return nil, fmt.Errorf("invalid URL format")
    }

    // Check scheme
    if parsed.Scheme != "http" && parsed.Scheme != "https" {
        return nil, fmt.Errorf("URL must use http or https")
    }

    // Check hostname exists
    if parsed.Hostname() == "" {
        return nil, fmt.Errorf("URL must have a hostname")
    }

    return parsed, nil
}
```

---

## Struct Validation

### Manual Validation

```go
type CreateUserRequest struct {
    Email    string `json:"email"`
    Name     string `json:"name"`
    Password string `json:"password"`
}

func (r *CreateUserRequest) Validate() error {
    // Email validation
    if r.Email == "" {
        return fmt.Errorf("email is required")
    }
    if !strings.Contains(r.Email, "@") {
        return fmt.Errorf("invalid email format")
    }

    // Name validation
    if r.Name == "" {
        return fmt.Errorf("name is required")
    }
    if len(r.Name) < 2 || len(r.Name) > 100 {
        return fmt.Errorf("name must be 2-100 characters")
    }

    // Password validation
    if r.Password == "" {
        return fmt.Errorf("password is required")
    }
    if len(r.Password) < 8 {
        return fmt.Errorf("password must be at least 8 characters")
    }

    return nil
}

// Usage:
func handleCreateUser(c *fiber.Ctx) error {
    var req CreateUserRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid JSON",
        })
    }

    if err := req.Validate(); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": err.Error(),
        })
    }

    // Proceed with creation...
}
```

### Using go-playground/validator

```go
import "github.com/go-playground/validator/v10"

type CreateUserRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Name     string `json:"name" validate:"required,min=2,max=100"`
    Password string `json:"password" validate:"required,min=8"`
}

var validate = validator.New()

func handleCreateUser(c *fiber.Ctx) error {
    var req CreateUserRequest
    if err := c.BodyParser(&req); err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid JSON",
        })
    }

    // Validate struct
    if err := validate.Struct(&req); err != nil {
        validationErrors := err.(validator.ValidationErrors)
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error":  "Validation failed",
            "fields": formatValidationErrors(validationErrors),
        })
    }

    // Proceed...
}

func formatValidationErrors(errs validator.ValidationErrors) map[string]string {
    errors := make(map[string]string)
    for _, err := range errs {
        errors[err.Field()] = fmt.Sprintf("failed validation: %s", err.Tag())
    }
    return errors
}
```

---

## Email Validation

```go
import "net/mail"

func validateEmail(email string) error {
    if email == "" {
        return fmt.Errorf("email is required")
    }

    // Use standard library email parser
    _, err := mail.ParseAddress(email)
    if err != nil {
        return fmt.Errorf("invalid email format")
    }

    return nil
}
```

---

## Sanitization

### String Sanitization

```go
import (
    "html"
    "strings"
)

func sanitizeString(input string) string {
    // Trim whitespace
    input = strings.TrimSpace(input)

    // Escape HTML to prevent XSS
    input = html.EscapeString(input)

    return input
}

// For database queries, always use parameterized queries
// Never sanitize and concat - use parameters instead!
```

### Filename Sanitization

```go
import (
    "path/filepath"
    "strings"
)

func sanitizeFilename(filename string) string {
    // Remove path components
    filename = filepath.Base(filename)

    // Remove dangerous characters
    filename = strings.Map(func(r rune) rune {
        switch {
        case r >= 'a' && r <= 'z':
            return r
        case r >= 'A' && r <= 'Z':
            return r
        case r >= '0' && r <= '9':
            return r
        case r == '.' || r == '-' || r == '_':
            return r
        default:
            return -1 // Remove character
        }
    }, filename)

    return filename
}
```

---

## Complete Validation Example

```go
func handleCreateUser(c *fiber.Ctx) error {
    // 1. Validate quality parameter
    quality, err := parseIntParam(c, "quality", 80, 1, 100)
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": err.Error(),
        })
    }

    // 2. Validate dimensions
    width, err := parseIntParam(c, "width", 0, 0, 10000)
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": err.Error(),
        })
    }

    height, err := parseIntParam(c, "height", 0, 0, 10000)
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": err.Error(),
        })
    }

    // 3. Validate format
    format := strings.ToLower(c.Query("format"))
    if format != "" {
        validFormats := []string{"jpeg", "png", "webp", "avif"}
        if err := validateStringParam(format, "format", validFormats); err != nil {
            return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
                "error": err.Error(),
            })
        }
    }

    // 4. Validate and read file
    imageData, err := validateImageFile(c, "image")
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": err.Error(),
        })
    }

    // All validation passed - proceed with processing
    result, err := services.CreateUser(imageData, services.OptimizeOptions{
        Quality: quality,
        Width:   width,
        Height:  height,
        Format:  format,
    })

    if err != nil {
        log.Printf("Error: %v", err)
        return c.Status(fiber.StatusInternalServerError).JSON(fiber.Map{
            "error": "Failed to process image",
        })
    }

    return c.JSON(result)
}
```

---

## Validation Best Practices

1. **Validate early** - Check input before any processing
2. **Be specific** - Provide clear error messages
3. **Set limits** - Always have min/max bounds
4. **Whitelist, don't blacklist** - Define what's allowed
5. **Check types** - Ensure correct data types
6. **Sanitize for display** - Escape HTML for XSS prevention
7. **Never sanitize for SQL** - Use parameterized queries instead
8. **Validate file types** - Check both extension and content
9. **Limit sizes** - Prevent resource exhaustion
10. **Log validation failures** - Track potential attacks

---

## Related Resources

- [security-guide.md](security-guide.md) - Security validation patterns
- [routing-and-handlers.md](routing-and-handlers.md) - Handler validation
- [error-handling.md](error-handling.md) - Validation error responses
