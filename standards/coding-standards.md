# Universal Coding Standards

These standards apply to all code regardless of language or framework. Language-specific standards in `skills/languages/` extend these.

---

## Principles

### 1. Readability Over Cleverness
Code is read far more than it is written. Prioritize clarity. Avoid "clever" tricks that require deep knowledge to understand. The next person to read this code may not be you.

### 2. Single Responsibility
Every function, method, class, and module should have one reason to change. If you can't explain what a piece of code does in one sentence, it does too much.

### 3. Fail Fast and Loudly
Detect errors early. Validate inputs at system boundaries. Fail with clear, actionable error messages. Never silently ignore errors or allow invalid state to propagate.

### 4. Don't Repeat Yourself (DRY) — With Judgment
Avoid duplicating logic when the things being duplicated are truly the same concept. However, don't abstract too early — duplication is better than a premature, wrong abstraction.

### 5. YAGNI — You Ain't Gonna Need It
Only build what is required now. Do not add complexity for hypothetical future requirements. Scope creep in code is as harmful as scope creep in requirements.

---

## Naming Standards

### General Rules

- Names should **reveal intent** — a name should tell you why something exists, what it does, and how it is used
- Names should be **pronounceable** — you should be able to say it in a code review
- Names should be **searchable** — avoid single-letter variables (except loop counters `i`, `j`) and magic numbers

### Functions and Methods

- Functions should be **verbs** or verb phrases: `calculateTotal`, `getUserByEmail`, `validateInput`
- Boolean functions should read as assertions: `isActive`, `hasPermission`, `canDelete`, `isEmpty`
- Functions that cause side effects should make that clear in their name: `sendEmail`, `writeToDatabase`, `invalidateCache`

### Classes and Types

- Classes should be **nouns** or noun phrases: `UserService`, `OrderProcessor`, `EmailValidator`
- Avoid vague suffixes: `Manager`, `Processor`, `Handler`, `Helper`, `Util`, `Data` — be specific about what the class actually does
- Exception: specific technical roles are fine: `UserRepository`, `UserMapper`, `UserValidator`

### Constants and Configuration

- Constants for magic values: `const TIMEOUT_MS = 5000` not `5000` scattered in code
- Avoid 0/1/true/false as function arguments without named parameters: `createUser(email, true)` is unclear; `createUser(email, { sendWelcomeEmail: true })` is not

---

## Function Design

### Size and Complexity

- **Target max function length:** 20–30 lines (soft limit — document if exceeded)
- **Target max cyclomatic complexity:** 10 (reduce with early returns or extraction)
- **Target max function parameters:** 4 (use an options/config object for more)

### Early Returns Over Nesting

```typescript
// ✅ Good — early return reduces nesting
function processOrder(order: Order): OrderResult {
  if (!order.items.length) return { error: 'Order has no items' };
  if (!order.customer) return { error: 'Order has no customer' };
  if (order.total <= 0) return { error: 'Order total must be positive' };

  // happy path — no nesting
  return executeOrder(order);
}

// ❌ Bad — deep nesting is hard to read
function processOrder(order: Order): OrderResult {
  if (order.items.length) {
    if (order.customer) {
      if (order.total > 0) {
        return executeOrder(order);
      } else {
        return { error: 'Order total must be positive' };
      }
    } else {
      return { error: 'Order has no customer' };
    }
  } else {
    return { error: 'Order has no items' };
  }
}
```

### Pure Functions Preferred

Favor functions without side effects that take input and return output. They are trivially testable and easy to reason about. Side effects (I/O, state mutation, logging, time) should be pushed to the edges of the system.

---

## Error Handling

### Never Swallow Errors

```typescript
// ❌ Never do this
try {
  await riskyOperation();
} catch (e) {
  // silently swallowed
}

// ❌ Never log-and-swallow without re-throwing or returning an error value
try {
  await riskyOperation();
} catch (e) {
  console.log('error occurred'); // and then what?
}

// ✅ Good — propagate with context
try {
  await riskyOperation();
} catch (err) {
  throw new ServiceError('Failed to execute risky operation', { cause: err });
}
```

### Error Messages Must Be Actionable

Good error message: "Email address is invalid. Expected format: user@domain.com"
Bad error message: "Validation failed"
Bad error message: "Error occurred"
Bad error message: "Something went wrong"

### Distinguish User-Facing and Internal Errors

- **User-facing errors** (validation, not found, forbidden): safe to return message to client
- **Internal errors** (database failure, unexpected state): log full details internally; return generic "internal error" message to client

---

## Code Organization

### Layer Responsibilities

| Layer | Responsibility | Should NOT contain |
|-------|---------------|-------------------|
| **Presentation** (controllers, handlers) | Parse request, validate input, call service, format response | Business logic, database queries |
| **Service / Application** | Business logic, orchestration | Database specifics, HTTP concepts |
| **Repository / Data Access** | Database queries, data mapping | Business rules, HTTP concepts |
| **Domain / Model** | Core entities, value objects, domain rules | Framework code, infrastructure |

### Dependency Direction

Dependencies must always point inward (toward domain):
```
Presentation → Service → Repository → Domain
                                    ↑
                          (no dependencies out)
```

---

## Comments and Documentation

### When to Comment

**Comment the "why", not the "what":**
```typescript
// ✅ Good — explains intent that isn't obvious from code
// The legacy API returns null for missing users but throws for deleted users.
// We normalize this to a consistent NotFoundError.
if (result === null || result.deletedAt !== null) {
  throw new NotFoundError('User', id);
}

// ❌ Bad — restates what the code says
// Get user by id
const user = await getUserById(id);
```

### When Not to Comment

- Do not comment self-explanatory code
- Do not leave commented-out code — use version control; delete it
- Do not leave `TODO` / `FIXME` without a linked issue and owner

### Documentation Required For

- Every public function, method, class, interface (use language-appropriate doc format)
- Every non-obvious algorithm or business rule
- Every configuration option and its valid values
- Every API endpoint (use OpenAPI)

---

## Configuration Standards

- No hard-coded values for URLs, timeouts, limits, feature flags, or environment-specific anything
- All configuration documented with: purpose, type, valid range/values, default, and whether it is required or optional
- Configuration validated at application startup — fail fast with a clear message if required config is missing
- Secrets are never configuration — use secrets management (see `skills/security/iam.instructions.md`)

---

## Dependency Standards

Before adding any new dependency:

1. **Is it necessary?** Can this be implemented without adding a dependency?
2. **Is it maintained?** Check GitHub activity — is it actively maintained?
3. **Is it secure?** Check for known CVEs; check it is from a trusted publisher
4. **Is it appropriately licensed?** GPL in a commercial product is a legal risk
5. **What is its size/impact?** Consider bundle size for frontend dependencies

Pin all dependency versions. Avoid version ranges that automatically upgrade to breaking or vulnerable versions:
```
// ❌ Bad — will auto-upgrade to potentially breaking or vulnerable version
"express": "^4.18.0"

// ✅ Good — deterministic; upgrade is a deliberate action
"express": "4.21.2"
```

---

## Performance Standards

- No premature optimization — profile first, optimize second
- Every database query must have an appropriate index — validate with `EXPLAIN ANALYZE` or equivalent
- Avoid N+1 queries — use joins, eager loading, or batch fetching
- Set timeouts on all external HTTP calls (default: 5 seconds; adjust per SLA)
- Set limits on all paginated queries (default max page size: 100 records)
- Cache aggressively at appropriate layers — but document what is cached, the TTL, and cache invalidation strategy
