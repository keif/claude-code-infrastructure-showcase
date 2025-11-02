# Services and Patterns

## Service Layer Design

### Service Structure

```go
// services/user_service.go
package services

type OptimizeOptions struct {
    Quality int
    Width   int
    Height  int
    Format  string
}

type OptimizeResult struct {
    OriginalSize  int64
    OptimizedSize int64
    Savings       string
}

func CreateUser(data []byte, opts OptimizeOptions) (*OptimizeResult, error) {
    // Business logic here
    return &OptimizeResult{}, nil
}
```

### Dependency Injection

```go
type ImageService struct {
    DB     *sql.DB
    Config *Config
}

func NewImageService(db *sql.DB, config *Config) *ImageService {
    return &ImageService{
        DB:     db,
        Config: config,
    }
}

func (s *ImageService) Optimize(data []byte) (*Result, error) {
    // Use s.DB and s.Config
    return &Result{}, nil
}
```
