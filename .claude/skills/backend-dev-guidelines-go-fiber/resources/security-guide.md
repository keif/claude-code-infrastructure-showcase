# Security Guide

## Security-First Development

Security is not an afterthought - it must be built into every layer of your API.

---

## SSRF (Server-Side Request Forgery) Protection

### The Threat

SSRF allows attackers to make the server fetch URLs they control, potentially accessing:
- Internal network resources (169.254.169.254 - cloud metadata)
- Localhost services
- Private IP ranges

### Protection Pattern

```go
// isPrivateIP checks if an IP is private/localhost/cloud metadata
func isPrivateIP(host string) bool {
    // Parse IP
    ip := net.ParseIP(host)
    if ip == nil {
        // Try to resolve hostname
        ips, err := net.LookupIP(host)
        if err != nil || len(ips) == 0 {
            return true // Block if can't resolve
        }
        ip = ips[0]
    }

    // Check for loopback (127.0.0.0/8, ::1)
    if ip.IsLoopback() {
        return true
    }

    // Check for private IP ranges (RFC 1918)
    if ip.IsPrivate() {
        return true
    }

    // Check for link-local (169.254.0.0/16)
    if ip.IsLinkLocalUnicast() {
        return true
    }

    // Block cloud metadata endpoints
    ipStr := ip.String()
    cloudMetadata := []string{
        "169.254.169.254", // AWS, Azure, GCP, OpenStack
        "fd00:ec2::254",   // AWS IPv6
    }
    for _, metadata := range cloudMetadata {
        if ipStr == metadata {
            return true
        }
    }

    return false
}

// Validate URL before fetching
func isAllowedDomain(u *url.URL) bool {
    hostname := u.Hostname()

    // Always block private IPs
    if isPrivateIP(hostname) {
        return false
    }

    // Check domain whitelist (if configured)
    if len(allowedDomains) > 0 {
        for _, domain := range allowedDomains {
            if hostname == domain || strings.HasSuffix(hostname, "."+domain) {
                return true
            }
        }
        return false // Not in whitelist
    }

    return true // No whitelist = allow all public domains
}
```

### Usage in Handler

```go
func handleFetchURL(c *fiber.Ctx) error {
    imgURL := c.FormValue("url")

    // Parse URL
    parsed, err := url.Parse(imgURL)
    if err != nil {
        return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
            "error": "Invalid URL format",
        })
    }

    // Check if allowed
    if !isAllowedDomain(parsed) {
        // SECURITY EVENT: Log the attempt
        log.Printf("[SECURITY] SSRF attempt blocked - IP: %s, URL: %s, Host: %s",
            c.IP(), imgURL, parsed.Hostname())

        return c.Status(fiber.StatusForbidden).JSON(fiber.Map{
            "error": "URL domain not allowed",
        })
    }

    // Safe to fetch
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()

    req, _ := http.NewRequestWithContext(ctx, "GET", imgURL, nil)
    resp, err := http.DefaultClient.Do(req)
    // ... handle response
}
```

---

## Decompression Bomb Protection

### The Threat

Small compressed files that expand to huge sizes, causing:
- Memory exhaustion
- Server crashes
- Denial of service

### Protection Pattern

```go
const maxDecodedPixels = 121_000_000 // 121 megapixels (11000x11000)

func validateDecodedImageSize(imgData []byte, filename string) error {
    // Get image dimensions without fully decoding
    metadata, err := bimg.NewImage(imgData).Metadata()
    if err != nil {
        return fmt.Errorf("failed to read metadata: %w", err)
    }

    // Calculate total pixels
    width := metadata.Size.Width
    height := metadata.Size.Height
    totalPixels := int64(width) * int64(height)

    // Check against limit
    if totalPixels > maxDecodedPixels {
        // SECURITY EVENT: Log the attempt
        log.Printf("[SECURITY] Decompression bomb - IP: %s, File: %s, Pixels: %d",
            clientIP, filename, totalPixels)

        return fmt.Errorf(
            "image '%s' is %dx%d pixels (%d total), exceeds max of %d pixels",
            filename, width, height, totalPixels, maxDecodedPixels,
        )
    }

    return nil
}
```

---

## API Key Authentication

### Pattern

```go
// middleware/api_key.go

type APIKeyConfig struct {
    Enabled        bool
    BypassRules    []BypassRule
    TrustedOrigins []string
}

type BypassRule struct {
    Path   string // Path prefix
    Method string // HTTP method (empty = all)
}

func RequireAPIKey() fiber.Handler {
    config := GetAPIKeyConfig()

    return func(c *fiber.Ctx) error {
        // Skip if disabled
        if !config.Enabled {
            return c.Next()
        }

        // Check trusted origins
        if isTrustedOrigin(c, config.TrustedOrigins) {
            return c.Next()
        }

        // Check bypass rules
        path := c.Path()
        method := c.Method()
        for _, rule := range config.BypassRules {
            if strings.HasPrefix(path, rule.Path) {
                if rule.Method == "" || rule.Method == method {
                    return c.Next()
                }
            }
        }

        // Extract API key
        authHeader := c.Get("Authorization")
        if authHeader == "" {
            log.Printf("[SECURITY] Missing API key - IP: %s, Path: %s",
                c.IP(), c.Path())
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "Missing API key",
            })
        }

        // Parse Bearer token
        apiKey := strings.TrimPrefix(authHeader, "Bearer ")

        // Validate
        if !db.ValidateAPIKey(apiKey) {
            keyPrefix := apiKey
            if len(keyPrefix) > 8 {
                keyPrefix = keyPrefix[:8] + "..."
            }
            log.Printf("[SECURITY] Invalid API key - IP: %s, Key: %s",
                c.IP(), keyPrefix)
            return c.Status(fiber.StatusUnauthorized).JSON(fiber.Map{
                "error": "Invalid or revoked API key",
            })
        }

        return c.Next()
    }
}
```

---

## Rate Limiting

### Token Bucket Pattern

```go
// middleware/rate_limit.go

type RateLimiter struct {
    visitors map[string]*Visitor
    mu       sync.RWMutex
}

type Visitor struct {
    limiter  *rate.Limiter
    lastSeen time.Time
}

func NewRateLimiter() fiber.Handler {
    rl := &RateLimiter{
        visitors: make(map[string]*Visitor),
    }

    // Cleanup old visitors every minute
    go rl.cleanupVisitors()

    return func(c *fiber.Ctx) error {
        ip := c.IP()

        // Get or create visitor
        limiter := rl.getVisitor(ip)

        if !limiter.Allow() {
            log.Printf("[SECURITY] Rate limit exceeded - IP: %s", ip)
            return c.Status(fiber.StatusTooManyRequests).JSON(fiber.Map{
                "error": "Rate limit exceeded",
            })
        }

        return c.Next()
    }
}

func (rl *RateLimiter) getVisitor(ip string) *rate.Limiter {
    rl.mu.Lock()
    defer rl.mu.Unlock()

    v, exists := rl.visitors[ip]
    if !exists {
        // Create new limiter: 10 requests per second, burst of 20
        limiter := rate.NewLimiter(10, 20)
        rl.visitors[ip] = &Visitor{limiter, time.Now()}
        return limiter
    }

    v.lastSeen = time.Now()
    return v.limiter
}
```

---

## Input Validation & Sanitization

### Parameter Validation

```go
// Always validate ALL input parameters

func parseQuality(c *fiber.Ctx) (int, error) {
    qualityStr := c.Query("quality")
    if qualityStr == "" {
        return 80, nil // Default
    }

    quality, err := strconv.Atoi(qualityStr)
    if err != nil {
        return 0, fmt.Errorf("quality must be a number")
    }

    // Validate range
    if quality < 1 || quality > 100 {
        return 0, fmt.Errorf("quality must be between 1 and 100")
    }

    return quality, nil
}

// In handler:
quality, err := parseQuality(c)
if err != nil {
    return c.Status(fiber.StatusBadRequest).JSON(fiber.Map{
        "error": err.Error(),
    })
}
```

### File Upload Validation

```go
const maxFileSize = 10 * 1024 * 1024 // 10MB

func validateUploadedFile(file *multipart.FileHeader) error {
    // Check size
    if file.Size > maxFileSize {
        return fmt.Errorf("file exceeds 10MB limit")
    }

    // Validate content type
    validTypes := map[string]bool{
        "image/jpeg": true,
        "image/png":  true,
        "image/webp": true,
    }

    contentType := file.Header.Get("Content-Type")
    if !validTypes[contentType] {
        return fmt.Errorf("invalid file type: %s", contentType)
    }

    return nil
}
```

---

## Security Headers

### Standard Security Headers

```go
// main.go - Add security headers middleware

app.Use(func(c *fiber.Ctx) error {
    // Prevent MIME sniffing
    c.Set("X-Content-Type-Options", "nosniff")

    // Prevent clickjacking
    c.Set("X-Frame-Options", "DENY")

    // XSS protection (legacy, but still useful)
    c.Set("X-XSS-Protection", "1; mode=block")

    // Referrer policy
    c.Set("Referrer-Policy", "strict-origin-when-cross-origin")

    // HSTS (only over HTTPS)
    if c.Protocol() == "https" {
        c.Set("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
    }

    return c.Next()
})
```

---

## CORS Configuration

### Secure CORS Setup

```go
// Custom CORS that checks against whitelist

func setupCORS(app *fiber.App) {
    allowedOrigins := os.Getenv("CORS_ORIGINS")
    if allowedOrigins == "" {
        allowedOrigins = "http://localhost:3000"
    }

    app.Use(func(c *fiber.Ctx) error {
        origin := c.Get("Origin")

        // Check if origin is allowed
        if origin != "" && strings.Contains(allowedOrigins, origin) {
            c.Set("Access-Control-Allow-Origin", origin)
            c.Set("Access-Control-Allow-Credentials", "true")
            c.Set("Access-Control-Allow-Headers", "Origin,Content-Type,Accept,Authorization")
            c.Set("Access-Control-Allow-Methods", "GET,POST,PUT,DELETE,OPTIONS")
        }

        // Handle preflight
        if c.Method() == "OPTIONS" {
            return c.SendStatus(fiber.StatusNoContent)
        }

        return c.Next()
    })
}
```

---

## Command Injection Prevention

### Safe Command Execution

```go
// When executing external commands, NEVER use user input directly

func optimizeWithExternalTool(inputBuffer []byte, level int) ([]byte, error) {
    // ✅ SAFE: Validate level is an integer in safe range
    if level < 0 || level > 6 {
        level = 2 // Safe default
    }

    // ✅ SAFE: Pass data via stdin/stdout, not file paths
    // ✅ SAFE: Parameters are hardcoded or validated integers
    cmd := exec.Command("oxipng",
        "-o", fmt.Sprintf("%d", level), // Validated integer
        "--strip", "all",                // Hardcoded string
        "--stdout",                      // Hardcoded flag
        "-",                             // Read from stdin
    )

    cmd.Stdin = bytes.NewReader(inputBuffer)
    var stdout bytes.Buffer
    cmd.Stdout = &stdout

    err := cmd.Run()
    if err != nil {
        return inputBuffer, err // Fallback to original
    }

    return stdout.Bytes(), nil
}

// ❌ UNSAFE: Never do this
func unsafeCommand(filename string) {
    // User-controlled filename in shell command - DANGEROUS!
    cmd := exec.Command("sh", "-c", "process "+filename)
    cmd.Run()
}
```

---

## SQL Injection Prevention

### Use Parameterized Queries

```go
// ✅ SAFE: Parameterized query
func FindUserByEmail(email string) (*User, error) {
    var user User
    err := DB.QueryRow(
        "SELECT id, email, name FROM users WHERE email = ?",
        email, // Parameter - safely escaped
    ).Scan(&user.ID, &user.Email, &user.Name)
    return &user, err
}

// ❌ UNSAFE: String concatenation
func unsafeQuery(email string) (*User, error) {
    query := "SELECT * FROM users WHERE email = '" + email + "'" // DANGEROUS!
    // email could be: "' OR '1'='1"
    rows, _ := DB.Query(query)
    // ...
}
```

---

## Environment Variable Security

### Never Commit Secrets

```bash
# .gitignore
.env
*.key
*.pem
credentials.json
config/secrets.yaml
```

### Load Secrets Safely

```go
// Use environment variables for all secrets
apiKey := os.Getenv("API_SECRET_KEY")
dbPassword := os.Getenv("DB_PASSWORD")

// Or use a secrets manager in production
// secret, err := secretsManager.GetSecret("prod/db/password")
```

---

## Logging Security

### Safe Logging Practices

```go
// ✅ DO: Log partial sensitive data
func logAPIKeyAttempt(apiKey string) {
    keyPrefix := apiKey
    if len(keyPrefix) > 8 {
        keyPrefix = keyPrefix[:8] + "..."
    }
    log.Printf("API key validation: %s", keyPrefix)
}

// ✅ DO: Log security events
log.Printf("[SECURITY] SSRF attempt blocked - IP: %s, URL: %s", ip, url)

// ❌ DON'T: Log full secrets
log.Printf("User authenticated with key: %s", apiKey) // BAD!

// ❌ DON'T: Log passwords ever
log.Printf("Login attempt: %s / %s", username, password) // NEVER!
```

---

## Security Checklist

- [ ] **SSRF Protection**: Validate all URLs, block private IPs
- [ ] **Decompression Bombs**: Limit decoded file sizes
- [ ] **API Key Auth**: Secure endpoints with authentication
- [ ] **Rate Limiting**: Prevent abuse
- [ ] **Input Validation**: Validate ALL parameters
- [ ] **Security Headers**: X-Content-Type-Options, X-Frame-Options, etc
- [ ] **CORS**: Whitelist allowed origins
- [ ] **Command Injection**: Use parameterized commands
- [ ] **SQL Injection**: Use parameterized queries
- [ ] **Secrets**: Never commit, use environment variables
- [ ] **Logging**: Don't log secrets, log security events
- [ ] **File Size Limits**: Prevent memory exhaustion
- [ ] **Timeouts**: Prevent resource exhaustion

---

## Related Resources

- [middleware-guide.md](middleware-guide.md) - Implementing security middleware
- [validation-patterns.md](validation-patterns.md) - Input validation
- [error-handling.md](error-handling.md) - Secure error responses
