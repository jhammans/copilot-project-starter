# Testing Standards

Testing is not optional. Untested code is not production-ready code.

---

## Testing Philosophy

**Tests are first-class code.** Apply the same naming, readability, and maintenance standards to tests as to production code.

Tests serve three purposes:
1. **Verification** — the code does what the requirements say it should
2. **Regression prevention** — future changes don't break existing behavior
3. **Documentation** — tests describe expected behavior in a way that code alone cannot

---

## Coverage Requirements

| Type | Minimum Line Coverage | Notes |
|------|--------------------|-------|
| Unit tests | 80% | Overall; measured per service/module |
| Security-critical code | 100% | Auth, authz, input validation, crypto |
| Business logic | 90% | Service layer |
| Infrastructure / adapters | 70% | Prefer integration tests over mocking |

Coverage is a **floor, not a target**. 100% coverage does not mean the tests are good; meaningful assertions matter more than line count.

---

## Test Types and When to Use Each

### Unit Tests

**What:** Test a single function, method, or class in isolation with all dependencies mocked.

**When:** For all business logic in the service/domain layer.

**What should be tested:**
- Happy path (valid inputs, expected outcome)
- Error cases (invalid inputs, missing required fields)
- Boundary conditions (empty arrays, zero values, max lengths)
- Business rule enforcement
- Security constraints (is authorization checked? Is input validated?)

```typescript
// ✅ Good unit test structure — Arrange / Act / Assert (AAA)
describe('UserService.createUser', () => {
  it('returns created user when input is valid', async () => {
    // Arrange
    const input = { email: 'test@example.com', password: 'SecurePass123!', firstName: 'Jan' };
    mockUserRepo.findByEmail.mockResolvedValue(null);
    mockUserRepo.insert.mockResolvedValue({ id: 'user-123', ...input });

    // Act
    const result = await userService.createUser(input);

    // Assert
    expect(result.id).toBe('user-123');
    expect(result.email).toBe('test@example.com');
    expect(mockUserRepo.insert).toHaveBeenCalledOnce();
  });

  it('throws ConflictError when email is already registered', async () => {
    // Arrange
    mockUserRepo.findByEmail.mockResolvedValue({ id: 'existing' });

    // Act + Assert
    await expect(
      userService.createUser({ email: 'taken@example.com', ... })
    ).rejects.toThrow(ConflictError);
  });
});
```

### Integration Tests

**What:** Test the interaction between components — typically a service + its real or in-memory database.

**When:** For every API endpoint and data access layer.

**Use Testcontainers** for database integration tests — run a real database, not an in-memory approximation:

```typescript
// Integration test with real database
describe('UsersApi POST /users', () => {
  let app: Express;
  let db: PrismaClient;

  beforeAll(async () => {
    db = await startTestDatabase(); // Testcontainers
    app = createApp({ db });
  });

  afterAll(async () => await stopTestDatabase());

  afterEach(async () => await db.user.deleteMany());

  it('creates a user and returns 201', async () => {
    const res = await request(app)
      .post('/api/v1/users')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ email: 'new@example.com', password: 'SecurePass123!', firstName: 'Jan', lastName: 'Doe' });

    expect(res.status).toBe(201);
    expect(res.body.email).toBe('new@example.com');
    expect(res.body.password).toBeUndefined(); // never return password

    // Verify persisted correctly
    const dbUser = await db.user.findUnique({ where: { email: 'new@example.com' } });
    expect(dbUser).not.toBeNull();
  });

  it('returns 400 for invalid email', async () => {
    const res = await request(app)
      .post('/api/v1/users')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({ email: 'not-an-email', password: 'SecurePass123!', firstName: 'Jan' });

    expect(res.status).toBe(400);
  });

  it('returns 401 when not authenticated', async () => {
    const res = await request(app)
      .post('/api/v1/users')
      .send({ email: 'new@example.com', ... });
    
    expect(res.status).toBe(401);
  });

  it('returns 403 when authenticated but lacks permission', async () => {
    const res = await request(app)
      .post('/api/v1/users')
      .set('Authorization', `Bearer ${readOnlyUserToken}`)
      .send({ email: 'new@example.com', ... });

    expect(res.status).toBe(403);
  });
});
```

### End-to-End (E2E) Tests

**What:** Test the full system from the user's perspective through the UI or API.

**When:** For critical user journeys (sign-up, login, primary feature workflows).

**Tools:** Playwright (preferred), Cypress.

**Scope:** Keep E2E tests focused on the most critical flows. Do not try to test every edge case with E2E tests — that's what unit and integration tests are for.

---

## Test Naming Standards

Test names must describe the expected behavior in plain English:

```
[unit]: [what is being tested] / [condition] / [expected outcome]

Examples:
"UserService.createUser / when email is already registered / throws ConflictError"
"PasswordValidator.validate / when password is less than 12 chars / returns false"
"GET /api/users/:id / when user does not exist / returns 404"
"GET /api/users/:id / when request is unauthenticated / returns 401"
```

---

## Security Test Coverage Requirements

Every API endpoint must have explicit tests for:

- [ ] **401 Unauthenticated** — request with no token returns 401
- [ ] **403 Forbidden** — authenticated user without required permission returns 403
- [ ] **IDOR check** — authenticated user cannot access another user's resource
- [ ] **Input validation** — malformed input returns 400, not 500
- [ ] **SQL injection probe** — inputs like `' OR 1=1 --` are handled safely (return 400, not 500 or data)

---

## Mocking Standards

- **Mock at the right layer:** mock external services (HTTP calls, email, SMS) — do not mock the thing you are testing
- **Do not mock the database in integration tests** — use a real test database
- **Verify behavior, not implementation:** assert on outcomes, not on how many times a private method was called
- **Reset mocks between tests** — prevent test pollution

```typescript
// ❌ Bad — testing implementation details, not behavior
expect(userRepository.insert).toHaveBeenCalledWith(
  expect.objectContaining({ passwordHash: expect.any(String) })
);

// ✅ Good — testing observable outcomes
const user = await userService.create({ email: 'test@example.com', password: 'pass' });
expect(user.passwordHash).not.toBe('pass'); // password was hashed
expect(user.email).toBe('test@example.com');
```

---

## Test Data Standards

- Use **factories or fixtures** for test data — do not scatter raw object literals across test files
- **Realistic but safe test data** — do not use real email domains or production user data in tests
- **Isolate test data** — each test should create and clean up its own data
- **Seed scripts** for integration test environments should be version-controlled

```typescript
// ✅ Good — test data factory
function buildUser(overrides: Partial<User> = {}): User {
  return {
    id: randomUUID(),
    email: `user.${Date.now()}@test.example.com`,
    firstName: 'Test',
    lastName: 'User',
    isActive: true,
    createdAt: new Date(),
    ...overrides,
  };
}
```

---

## Performance Testing (Required for APIs Deployed to Production)

Before any API is deployed to production, it must have a baseline performance profile:

- **Load test** at 2× expected p95 concurrent users — no errors, latency within SLA
- **Stress test** to determine breaking point — results documented
- **Database query analysis** — all queries analyzed with `EXPLAIN` for efficiency

Tools: k6, Artillery, Locust, or JMeter.

---

## CI/CD Pipeline Test Requirements

Every CI pipeline must:

1. Run all unit tests on every commit — fail PR if any test fails
2. Run integration tests on every commit or at minimum on PRs to main/develop
3. Run coverage report — fail if coverage drops below minimum thresholds
4. Run security tests (SAST) — fail if critical findings
5. Run dependency vulnerability scan — fail if known CVEs above medium severity
