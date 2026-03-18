---
applyTo: "**/*.py"
---

# Python Coding Standards

Apply these standards to all Python code.

---

## Python Version & Toolchain

- Minimum Python version: **3.11+**
- Package manager: **uv** (preferred) or pip with `requirements.txt` pinned to exact versions
- Type checker: **mypy** with strict mode
- Linter: **ruff** (replaces flake8, pylint, isort, black)
- Formatter: **ruff format** (replaces black)

### `pyproject.toml` — Required Configuration

```toml
[tool.mypy]
python_version = "3.11"
strict = true
disallow_any_generics = true
disallow_untyped_decorators = true
warn_return_any = true
warn_unused_configs = true

[tool.ruff]
target-version = "py311"
line-length = 100

[tool.ruff.lint]
select = ["E", "F", "I", "N", "S", "UP", "B", "A", "C4", "T20"]
ignore = []

[tool.ruff.lint.per-file-ignores]
"tests/**" = ["S101"]  # Allow assert in tests
```

---

## Type Annotations

All functions and methods must have complete type annotations.

```python
# ✅ Good — fully typed
from typing import Optional
from collections.abc import Sequence

def get_user_by_email(email: str) -> Optional["User"]:
    ...

def calculate_total(items: Sequence[float], discount: float = 0.0) -> float:
    ...

# ✅ Good — use | syntax (Python 3.10+)
def process(value: str | None) -> dict[str, str]:
    ...

# ❌ Bad — no annotations
def get_user(email):
    ...
```

### Avoid `Any` Type

```python
from typing import Any  # use sparingly
from typing import Unknown  # prefer for truly unknown types

# ✅ If Any is unavoidable, document why
data: Any  # External library returns untyped dict; tracked in ISSUE-456
```

---

## Naming Conventions

| Construct | Convention | Example |
|-----------|-----------|---------|
| Variables & functions | `snake_case` | `get_user_by_id` |
| Classes | `PascalCase` | `UserService`, `DatabaseError` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Private attributes | `_leading_underscore` | `_cache`, `_db_connection` |
| Type aliases | `PascalCase` | `UserId = str` |
| Files / modules | `snake_case` | `user_service.py` |
| Packages | `snake_case` | `auth_utils/` |

---

## Project Structure

```
src/
├── __init__.py
├── main.py               # Application entry point
├── config.py             # Configuration loading (pydantic-settings preferred)
├── errors.py             # Custom exception hierarchy
├── models/               # Pydantic models / domain entities
├── services/             # Business logic
├── repositories/         # Data access layer
├── api/                  # API routes (FastAPI routers or Flask blueprints)
│   ├── __init__.py
│   ├── dependencies.py   # FastAPI deps / request guards
│   └── v1/
└── utils/                # Pure utility functions

tests/
├── conftest.py
├── unit/
└── integration/
```

---

## Error Handling

```python
# ✅ Good — custom exception hierarchy
class AppError(Exception):
    """Base exception for all application errors."""
    pass

class NotFoundError(AppError):
    def __init__(self, resource: str, id: str) -> None:
        super().__init__(f"{resource} with id '{id}' not found")
        self.resource = resource
        self.id = id

class ValidationError(AppError):
    def __init__(self, field: str, message: str) -> None:
        super().__init__(f"Validation failed for '{field}': {message}")
        self.field = field

# ✅ Good — never swallow exceptions silently
try:
    user = db.get_user(user_id)
except DatabaseError as exc:
    logger.error("Failed to fetch user %s", user_id, exc_info=exc)
    raise ServiceError("Unable to retrieve user") from exc

# ❌ Bad — silently swallowing
try:
    result = risky_operation()
except Exception:
    pass  # NEVER
```

---

## Security Patterns

### Input Validation with Pydantic

```python
from pydantic import BaseModel, EmailStr, Field, field_validator
import re

class CreateUserRequest(BaseModel):
    email: EmailStr
    password: str = Field(min_length=12, max_length=128)
    first_name: str = Field(min_length=1, max_length=100)
    last_name: str = Field(min_length=1, max_length=100)

    @field_validator("first_name", "last_name")
    @classmethod
    def sanitize_name(cls, v: str) -> str:
        # Strip leading/trailing whitespace
        return v.strip()
```

### Configuration with pydantic-settings

```python
from pydantic_settings import BaseSettings
from pydantic import SecretStr, AnyHttpUrl

class Settings(BaseSettings):
    database_url: SecretStr
    jwt_secret: SecretStr = Field(min_length=32)
    debug: bool = False
    allowed_origins: list[AnyHttpUrl] = []

    class Config:
        env_file = ".env"  # for local dev only

settings = Settings()  # Validates and loads at startup; fails fast if missing
```

### Never Use `eval`, `exec`, or `pickle` with User Data

```python
# ❌ Never
eval(user_input)          # Code injection
exec(user_input)          # Code injection
pickle.loads(user_data)   # Deserialization attack

# ✅ Use safe alternatives
import ast
ast.literal_eval(user_input)  # Only for Python literals (str, int, dict, list, etc.)
import json
json.loads(user_input)         # For JSON data
```

### SQL Queries — Always Parameterized

```python
# ✅ Good
async def get_user(email: str) -> User | None:
    row = await db.fetchrow(
        "SELECT * FROM users WHERE email = $1 AND active = TRUE",
        email  # parameterized
    )
    return User(**row) if row else None

# ❌ Bad
query = f"SELECT * FROM users WHERE email = '{email}'"  # SQL injection
```

---

## Logging Standards

```python
import logging
import structlog  # preferred for structured JSON logging

logger = structlog.get_logger(__name__)

# ✅ Good — structured logging with context
logger.info(
    "User authenticated",
    user_id=user.id,
    event="auth.login.success",
    ip=request.client.host,
)

# ✅ Good — standard logging
logger.info("User %s authenticated from %s", user.id, request.client.host)

# ❌ Never log PII
logger.info("Login: email=%s password=%s", email, password)  # NEVER log passwords
logger.info("User data: %s", user.__dict__)  # may contain PII
```

---

## Docstrings (Google Style)

```python
def calculate_discount(price: float, discount_pct: float) -> float:
    """Calculate the discounted price.

    Args:
        price: Original price in dollars. Must be non-negative.
        discount_pct: Discount percentage as a decimal (0.0–1.0).

    Returns:
        The price after applying the discount.

    Raises:
        ValueError: If price is negative or discount_pct is outside [0.0, 1.0].

    Example:
        >>> calculate_discount(100.0, 0.2)
        80.0
    """
    if price < 0:
        raise ValueError(f"Price must be non-negative, got {price}")
    if not 0.0 <= discount_pct <= 1.0:
        raise ValueError(f"Discount must be in [0.0, 1.0], got {discount_pct}")
    return price * (1 - discount_pct)
```

All public modules, classes, functions, and methods must have docstrings.
