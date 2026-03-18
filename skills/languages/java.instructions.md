---
applyTo: "**/*.java"
---

# Java Coding Standards

Apply these standards to all Java code. Assumes Java 17 LTS or later.

---

## Build & Toolchain

- Build tool: **Maven** or **Gradle** (Kotlin DSL preferred for Gradle)
- Java version: **21 LTS** (minimum 17)
- Code formatter: **google-java-format** or **Palantir Java Format**
- Static analysis: **SpotBugs**, **PMD**, **Checkstyle**
- Dependency vulnerability scanning: **OWASP Dependency Check**

### Dependency Management
- Pin all dependency versions — no version ranges in production (`[1.0,)` is prohibited)
- Use a BOM (Bill of Materials) for framework family versions (Spring BOM, etc.)
- Review security advisories before accepting any new dependency
- Run `./gradlew dependencyCheckAnalyze` in CI for every PR

---

## Code Style

### Naming Conventions

| Construct | Convention | Example |
|-----------|-----------|---------|
| Classes & interfaces | `PascalCase` | `UserService`, `AuthenticationProvider` |
| Methods & variables | `camelCase` | `getUserById`, `isActive` |
| Constants | `SCREAMING_SNAKE_CASE` | `MAX_RETRIES`, `DEFAULT_TIMEOUT_MS` |
| Packages | `lowercase.dotted` | `com.example.users.service` |
| Test classes | `SUT + Test` | `UserServiceTest` |

### Package Structure

```
com.example.myapp/
├── config/           # Spring configuration
├── domain/           # Domain entities, value objects
├── application/      # Use cases / application services
├── infrastructure/   # Repositories, external service clients
├── api/              # Controllers (REST), DTOs, mappers
│   ├── v1/
│   └── common/
├── security/         # Security config, filters, JWT utilities
└── shared/           # Shared utilities, exceptions
```

---

## Modern Java Practices

### Use Records for Immutable Data

```java
// ✅ Good — immutable data carrier
public record CreateUserRequest(
    @NotBlank @Email String email,
    @NotBlank @Size(min = 12, max = 128) String password,
    @NotBlank @Size(max = 100) String firstName,
    @NotBlank @Size(max = 100) String lastName
) {}

// ✅ Good — domain value object
public record UserId(UUID value) {
    public UserId {
        Objects.requireNonNull(value, "UserId cannot be null");
    }
    public static UserId generate() {
        return new UserId(UUID.randomUUID());
    }
}
```

### Use Sealed Classes for Domain Modeling

```java
// ✅ Good — exhaustive result type
public sealed interface Result<T> permits Result.Success, Result.Failure {
    record Success<T>(T value) implements Result<T> {}
    record Failure<T>(String message, Throwable cause) implements Result<T> {}
}
```

### Prefer Optional Correctly

```java
// ✅ Good — Optional as return type for nullable results
public Optional<User> findUserByEmail(String email) {
    return userRepository.findByEmail(email);
}

// ❌ Bad — Optional in fields or parameters
private Optional<String> name; // use @Nullable instead
void process(Optional<String> value) {} // never as parameter
```

---

## Exception Handling

```java
// ✅ Good — custom exception hierarchy
public abstract class AppException extends RuntimeException {
    protected AppException(String message) { super(message); }
    protected AppException(String message, Throwable cause) { super(message, cause); }
}

public class NotFoundException extends AppException {
    private final String resource;
    private final String id;

    public NotFoundException(String resource, String id) {
        super(String.format("%s with id '%s' not found", resource, id));
        this.resource = resource;
        this.id = id;
    }
}

// ✅ Good — catch specific exceptions; add context; never swallow
try {
    return userRepository.findById(id)
        .orElseThrow(() -> new NotFoundException("User", id.toString()));
} catch (DataAccessException ex) {
    log.error("Database error fetching user {}", id, ex);
    throw new ServiceException("Failed to retrieve user", ex);
}

// ❌ Bad — catching Throwable or Exception without re-throwing
try {
    ...
} catch (Exception e) {
    // silently swallowed — NEVER
}
```

---

## Security Patterns

### Input Validation (Bean Validation)

```java
@RestController
@RequestMapping("/api/users")
@Validated
public class UserController {

    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @Valid @RequestBody CreateUserRequest request) {
        // request is already validated by @Valid
        User user = userService.create(request);
        return ResponseEntity.status(201).body(UserMapper.toResponse(user));
    }
}
```

### Never Use String Concatenation for Queries

```java
// ✅ Good — JPA/Spring Data
Optional<User> findByEmailAndActiveTrue(String email);

// ✅ Good — JPQL with named parameter
@Query("SELECT u FROM User u WHERE u.email = :email AND u.active = TRUE")
Optional<User> findActiveUserByEmail(@Param("email") String email);

// ✅ Good — JDBC with parameterized query
jdbcTemplate.query(
    "SELECT * FROM users WHERE email = ? AND active = TRUE",
    userRowMapper,
    email  // parameter — not concatenated
);

// ❌ Bad — SQL injection
String query = "SELECT * FROM users WHERE email = '" + email + "'"; // NEVER
```

### Never Log Sensitive Data

```java
// ✅ Good
log.info("User {} authenticated from {}", user.getId(), clientIp);

// ❌ Bad
log.info("Login attempt: email={}, password={}", email, password); // NEVER
log.debug("User object: {}", user); // user.toString() may expose PII
```

### Configuration — Never Hard-Code Secrets

```java
// ✅ Good — externalized configuration
@ConfigurationProperties(prefix = "app.security")
@Validated
public record SecurityProperties(
    @NotBlank String jwtSecret,
    @Positive int jwtExpirationSeconds
) {}

// application.yml — values from environment variables, not hardcoded
app:
  security:
    jwt-secret: ${JWT_SECRET}
    jwt-expiration-seconds: ${JWT_EXPIRATION_SECONDS:900}
```

---

## Javadoc Standards

All public classes, interfaces, and methods must have Javadoc:

```java
/**
 * Retrieves a user by their email address.
 *
 * <p>Only active (non-deleted, non-suspended) users are returned.
 *
 * @param email the user's email address; must be a valid email format
 * @return an {@link Optional} containing the user if found and active,
 *         or empty if no such user exists
 * @throws ValidationException if {@code email} is null or blank
 * @throws ServiceException if a database error occurs
 */
public Optional<User> findActiveUserByEmail(String email) { ... }
```

---

## Testing Standards

- Test framework: **JUnit 5** with **AssertJ** for assertions
- Mocking: **Mockito**
- Integration tests: **Spring Boot Test** + **Testcontainers** for real databases
- Test coverage: minimum 80% line coverage; 100% on security-critical code

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @InjectMocks
    private UserService userService;

    @Test
    @DisplayName("findActiveUserByEmail returns user when found and active")
    void findActiveUserByEmail_whenFound_returnsUser() {
        // Given
        var email = "test@example.com";
        var expectedUser = UserTestFixtures.activeUser(email);
        when(userRepository.findByEmailAndActiveTrue(email))
            .thenReturn(Optional.of(expectedUser));

        // When
        var result = userService.findActiveUserByEmail(email);

        // Then
        assertThat(result).isPresent().contains(expectedUser);
    }

    @Test
    @DisplayName("findActiveUserByEmail returns empty when user not found")
    void findActiveUserByEmail_whenNotFound_returnsEmpty() {
        // Given
        when(userRepository.findByEmailAndActiveTrue(any()))
            .thenReturn(Optional.empty());

        // When
        var result = userService.findActiveUserByEmail("missing@example.com");

        // Then
        assertThat(result).isEmpty();
    }
}
```
