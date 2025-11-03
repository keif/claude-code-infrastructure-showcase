# Testing Guide

## Go Testing Patterns

### Unit Test Structure

```go
func TestCreateUser(t *testing.T) {
    // Setup
    testData := loadTestData()
    options := UserOptions{Quality: 80}

    // Execute
    result, err := CreateUser(testData, options)

    // Assert
    if err != nil {
        t.Fatalf("Expected no error, got: %v", err)
    }

    if result.ProcessedSize >= result.OriginalSize {
        t.Errorf("Expected size reduction, got %d >= %d",
            result.ProcessedSize, result.OriginalSize)
    }
}
```

### HTTP Handler Testing

```go
func TestHandleCreateUser(t *testing.T) {
    app := fiber.New()
    app.Post("/users", handleCreateUser)

    req := httptest.NewRequest("POST", "/users", nil)
    resp, err := app.Test(req)

    if err != nil {
        t.Fatal(err)
    }

    if resp.StatusCode != 200 {
        t.Errorf("Expected 200, got %d", resp.StatusCode)
    }
}
```

### Table-Driven Tests

```go
func TestValidateQuality(t *testing.T) {
    tests := []struct {
        name    string
        input   int
        wantErr bool
    }{
        {"valid quality", 80, false},
        {"too low", 0, true},
        {"too high", 101, true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := validateQuality(tt.input)
            if (err != nil) != tt.wantErr {
                t.Errorf("got error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```
