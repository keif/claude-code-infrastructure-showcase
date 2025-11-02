# Database Patterns

## database/sql Best Practices

### Connection Setup

```go
import (
    "database/sql"
    _ "github.com/mattn/go-sqlite3"
)

func Initialize() error {
    db, err := sql.Open("sqlite3", "./data/app.db")
    if err != nil {
        return err
    }

    // Test connection
    if err := db.Ping(); err != nil {
        return err
    }

    DB = db
    return createSchema()
}
```

### Parameterized Queries

```go
// ✅ SAFE: Parameterized query
func FindUser(id string) (*User, error) {
    var user User
    err := DB.QueryRow(
        "SELECT id, email, name FROM users WHERE id = ?",
        id,
    ).Scan(&user.ID, &user.Email, &user.Name)
    return &user, err
}
```

### Transactions

```go
func TransferFunds(fromID, toID string, amount int) error {
    tx, err := DB.Begin()
    if err != nil {
        return err
    }
    defer tx.Rollback() // Rollback if not committed

    // Debit
    _, err = tx.Exec("UPDATE accounts SET balance = balance - ? WHERE id = ?", amount, fromID)
    if err != nil {
        return err
    }

    // Credit
    _, err = tx.Exec("UPDATE accounts SET balance = balance + ? WHERE id = ?", amount, toID)
    if err != nil {
        return err
    }

    return tx.Commit()
}
```
