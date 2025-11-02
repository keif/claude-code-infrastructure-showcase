# Configuration

## Environment Variables

### Loading Configuration

```go
import "os"

type Config struct {
    Port     string
    DBPath   string
    LogLevel string
}

func LoadConfig() *Config {
    return &Config{
        Port:     getEnv("PORT", "8080"),
        DBPath:   getEnv("DB_PATH", "./data/app.db"),
        LogLevel: getEnv("LOG_LEVEL", "info"),
    }
}

func getEnv(key, defaultValue string) string {
    if value := os.Getenv(key); value != "" {
        return value
    }
    return defaultValue
}
```

### Environment Files

```bash
# .env
PORT=8080
DB_PATH=./data/production.db
LOG_LEVEL=info
API_KEY_SECRET=your-secret-here
```

### Loading .env Files

```go
import "github.com/joho/godotenv"

func init() {
    godotenv.Load() // Load .env file
}
```
