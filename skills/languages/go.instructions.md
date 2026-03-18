---
applyTo: "**/*.go"
---

# Go Coding Standards

Apply these standards to all Go code. Assumes Go 1.22 or later.

---

## Toolchain

- Go version: **1.22+**
- Linter: **golangci-lint** (with configured set of linters)
- Security scanner: **gosec**
- Vulnerability check: `govulncheck ./...`
- Module management: Go modules with `go.sum` checked into source control

### `.golangci.yml` — Required Configuration

```yaml
run:
  timeout: 5m

linters:
  enable:
    - errcheck
    - gosimple
    - govet
    - ineffassign
    - staticcheck
    - unused
    - gosec
    - gocritic
    - revive
    - gofmt
    - goimports
    - noctx        # HTTP requests with context
    - bodyclose    # response body closed

linters-settings:
  gosec:
    excludes: []  # do not exclude any security checks
```

---

## Project Structure

Follow the standard Go project layout:

```
myapp/
├── cmd/
│   └── server/
│       └── main.go        # Entry point; minimal code — delegates to internal
├── internal/              # Private application code; not importable by other modules
│   ├── config/
│   ├── domain/            # Domain entities and interfaces
│   ├── service/           # Business logic
│   ├── repository/        # Data access
│   ├── handler/           # HTTP handlers
│   └── middleware/        # HTTP middleware
├── pkg/                   # Public library code (only if others will import it)
├── api/                   # OpenAPI/Protobuf spec files
├── migrations/            # Database migration files
└── go.mod
```

---

## Error Handling

Go's explicit error handling is a feature — respect it.

```go
// ✅ Good — always check and handle errors
user, err := userRepo.FindByEmail(ctx, email)
if err != nil {
    if errors.Is(err, ErrNotFound) {
        return nil, fmt.Errorf("user not found: %w", err)
    }
    return nil, fmt.Errorf("finding user by email: %w", err)
}

// ✅ Good — define sentinel errors
var (
    ErrNotFound   = errors.New("not found")
    ErrForbidden  = errors.New("forbidden")
    ErrConflict   = errors.New("conflict")
)

// ✅ Good — typed errors for structured data
type ValidationError struct {
    Field   string
    Message string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation error: field '%s': %s", e.Field, e.Message)
}

// ❌ Bad — ignoring errors
user, _ := userRepo.FindByEmail(ctx, email) // NEVER ignore errors

// ❌ Bad — using panic for control flow
func getUser(id string) *User {
    user, err := db.FindUser(id)
    if err != nil {
        panic(err) // only panic for truly unrecoverable programmer errors
    }
    return user
}
```

---

## Naming Conventions

| Construct | Convention | Example |
|-----------|-----------|---------|
| Packages | short `lowercase` | `user`, `auth`, `config` |
| Exported types & functions | `PascalCase` | `UserService`, `GetUserByID` |
| Unexported types & functions | `camelCase` | `parseToken`, `userCache` |
| Interfaces | noun or noun phrase | `UserRepository`, `TokenValidator` |
| Error variables | `Err` prefix | `ErrNotFound`, `ErrConflict` |
| Constants | `PascalCase` (exported) or `camelCase` (unexported) | `MaxRetries` |
| Context parameter | always first: `ctx context.Context` | `func Get(ctx context.Context, id string)` |

---

## Context Usage

```go
// ✅ Good — context first in every function that does I/O
func (s *UserService) CreateUser(ctx context.Context, req CreateUserRequest) (*User, error) {
    if err := s.validator.Validate(req); err != nil {
        return nil, fmt.Errorf("validating create user request: %w", err)
    }
    return s.repo.Insert(ctx, req)
}

// ✅ Good — pass request context through the call chain
func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context() // use request context; it carries trace IDs, deadlines
    user, err := h.userService.CreateUser(ctx, req)
    ...
}

// ❌ Bad — using context.Background() in a handler (loses trace context, deadlines)
user, err := h.userService.CreateUser(context.Background(), req)
```

---

## Security Patterns

### Input Validation

```go
// ✅ Use a validation library (go-playground/validator) or write explicit checks
type CreateUserRequest struct {
    Email     string `json:"email"     validate:"required,email,max=255"`
    Password  string `json:"password"  validate:"required,min=12,max=128"`
    FirstName string `json:"firstName" validate:"required,max=100"`
    LastName  string `json:"lastName"  validate:"required,max=100"`
}

func (h *UserHandler) CreateUser(w http.ResponseWriter, r *http.Request) {
    var req CreateUserRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    if err := h.validator.Struct(req); err != nil {
        http.Error(w, "Validation failed", http.StatusBadRequest)
        return
    }
    ...
}
```

### SQL — Always Parameterized

```go
// ✅ Good — parameterized query with database/sql
row := db.QueryRowContext(ctx,
    "SELECT id, email FROM users WHERE email = $1 AND active = TRUE",
    email,  // parameter
)

// ✅ Good — sqlc generated code (all queries parameterized at compile time)
user, err := queries.GetActiveUserByEmail(ctx, email)

// ❌ Bad — string construction
query := fmt.Sprintf("SELECT * FROM users WHERE email = '%s'", email) // SQL injection
```

### Never Use `os/exec` with User Input

```go
// ❌ Bad — command injection
cmd := exec.Command("sh", "-c", "convert "+userInput)

// ✅ Good — explicit args, no shell interpolation
cmd := exec.CommandContext(ctx, "convert",
    "-resize", "800x600",
    sanitizedInputPath,
    outputPath,
)
```

### Configuration — No Hard-Coded Secrets

```go
// ✅ Good — load from environment
type Config struct {
    DatabaseURL string
    JWTSecret   string
    Port        int
}

func LoadConfig() (*Config, error) {
    cfg := &Config{
        DatabaseURL: os.Getenv("DATABASE_URL"),
        JWTSecret:   os.Getenv("JWT_SECRET"),
        Port:        3000,
    }
    
    if cfg.DatabaseURL == "" {
        return nil, errors.New("DATABASE_URL environment variable is required")
    }
    if len(cfg.JWTSecret) < 32 {
        return nil, errors.New("JWT_SECRET must be at least 32 characters")
    }
    
    if portStr := os.Getenv("PORT"); portStr != "" {
        port, err := strconv.Atoi(portStr)
        if err != nil {
            return nil, fmt.Errorf("invalid PORT value: %w", err)
        }
        cfg.Port = port
    }
    
    return cfg, nil
}
```

---

## HTTP Handler Patterns

```go
// ✅ Good — structured handler with proper response writing
func (h *UserHandler) GetUser(w http.ResponseWriter, r *http.Request) {
    ctx := r.Context()
    userID := r.PathValue("id") // Go 1.22 built-in path params

    user, err := h.userService.GetByID(ctx, userID)
    if err != nil {
        switch {
        case errors.Is(err, ErrNotFound):
            writeError(w, http.StatusNotFound, "User not found")
        case errors.Is(err, ErrForbidden):
            writeError(w, http.StatusForbidden, "Access denied")
        default:
            h.logger.ErrorContext(ctx, "Failed to get user",
                slog.String("userID", userID),
                slog.Any("error", err),
            )
            writeError(w, http.StatusInternalServerError, "Internal server error")
        }
        return
    }

    writeJSON(w, http.StatusOK, user)
}
```

---

## GoDoc Standards

Every exported package, type, function, and method needs a documentation comment:

```go
// Package user provides services for managing user accounts.
package user

// UserService handles business logic for user management including
// creation, authentication, and profile updates.
type UserService struct { ... }

// GetByID retrieves an active user by their unique identifier.
//
// It returns ErrNotFound if no active user with the given id exists.
// It wraps any database errors with additional context.
func (s *UserService) GetByID(ctx context.Context, id string) (*User, error) { ... }
```

---

## Testing Standards

- Use the standard `testing` package
- Table-driven tests for multiple cases
- Testify for assertions: `github.com/stretchr/testify`
- HTTP handler tests: `httptest.NewRecorder` + `httptest.NewRequest`
- Coverage: `go test -cover ./...` — minimum 80%

```go
func TestUserService_GetByID(t *testing.T) {
    tests := []struct {
        name      string
        id        string
        mockSetup func(*MockUserRepository)
        want      *User
        wantErr   error
    }{
        {
            name: "returns user when found",
            id:   "user-123",
            mockSetup: func(m *MockUserRepository) {
                m.On("FindByID", mock.Anything, "user-123").
                    Return(&User{ID: "user-123"}, nil)
            },
            want: &User{ID: "user-123"},
        },
        {
            name: "returns ErrNotFound when user does not exist",
            id:   "missing-id",
            mockSetup: func(m *MockUserRepository) {
                m.On("FindByID", mock.Anything, "missing-id").
                    Return(nil, ErrNotFound)
            },
            wantErr: ErrNotFound,
        },
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            repo := new(MockUserRepository)
            tt.mockSetup(repo)
            svc := NewUserService(repo)

            got, err := svc.GetByID(context.Background(), tt.id)

            if tt.wantErr != nil {
                assert.ErrorIs(t, err, tt.wantErr)
            } else {
                require.NoError(t, err)
                assert.Equal(t, tt.want, got)
            }
        })
    }
}
```
