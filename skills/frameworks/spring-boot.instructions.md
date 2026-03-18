---
applyTo: "**/*.java,**/spring/**,**/src/main/java/**"
---

# Spring Boot Coding Standards

Apply these standards to all Spring Boot applications. Assumes Spring Boot 3.x with Java 21.

---

## Dependencies (Always Use Latest Patch in These Families)

```xml
<!-- pom.xml (Maven) -->
<parent>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-parent</artifactId>
  <version>3.4.x</version>
</parent>
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
</dependencies>
```

---

## Project Structure

```
src/main/java/com/example/
├── Application.java           # @SpringBootApplication entry point
├── config/                    # Configuration classes
│   ├── SecurityConfig.java
│   ├── WebMvcConfig.java
│   └── OpenApiConfig.java
├── domain/                    # Domain entities, value objects
│   ├── model/
│   └── event/
├── application/               # Application services / use cases
│   └── UserService.java
├── infrastructure/            # External adapters
│   ├── persistence/           # JPA repositories and entities
│   ├── rest/client/           # HTTP clients for external APIs
│   └── messaging/             # Kafka/RabbitMQ producers/consumers
├── api/                       # REST layer
│   ├── v1/
│   │   ├── UserController.java
│   │   └── dto/
│   │       ├── CreateUserRequest.java
│   │       └── UserResponse.java
│   └── common/
│       ├── ApiError.java
│       └── GlobalExceptionHandler.java
└── security/                  # Security utilities
    ├── JwtAuthenticationFilter.java
    └── UserDetailsServiceImpl.java
```

---

## Security Configuration

### Spring Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)  // enables @PreAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(
            HttpSecurity http,
            JwtAuthenticationFilter jwtFilter
    ) throws Exception {
        return http
            // Disable CSRF for stateless REST APIs using JWT
            .csrf(AbstractHttpConfigurer::disable)
            // Stateless session — no HttpSession
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            // Security headers
            .headers(h -> h
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
                .frameOptions(fo -> fo.deny())
                .xssProtection(xss -> xss.disable()) // deprecated; use CSP instead
            )
            // Access rules
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .exceptionHandling(e -> e
                .authenticationEntryPoint(new BearerTokenAuthEntryPoint())
                .accessDeniedHandler(new CustomAccessDeniedHandler())
            )
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        // bcrypt with cost factor 12 (increase if hardware allows)
        return new BCryptPasswordEncoder(12);
    }
}
```

### Method-Level Security

```java
@RestController
@RequestMapping("/api/v1/documents")
@RequiredArgsConstructor
public class DocumentController {

    private final DocumentService documentService;

    @GetMapping
    @PreAuthorize("hasAuthority('documents:read')")
    public ResponseEntity<Page<DocumentResponse>> list(Pageable pageable) {
        return ResponseEntity.ok(documentService.list(pageable));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('documents:delete')")
    public ResponseEntity<Void> delete(@PathVariable UUID id) {
        documentService.delete(id);
        return ResponseEntity.noContent().build();
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasAuthority('documents:update') and @documentAcl.isOwner(#id, authentication)")
    public ResponseEntity<DocumentResponse> update(
            @PathVariable UUID id,
            @Valid @RequestBody UpdateDocumentRequest request
    ) {
        return ResponseEntity.ok(documentService.update(id, request));
    }
}
```

---

## Exception Handling

```java
// api/common/GlobalExceptionHandler.java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(NotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(NotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ApiError.of("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fe -> fe.getDefaultMessage() != null ? fe.getDefaultMessage() : "Invalid value"
            ));
        return ResponseEntity.badRequest()
            .body(ApiError.validation(errors));
    }

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ApiError> handleAccessDenied(AccessDeniedException ex, HttpServletRequest req) {
        log.warn("Access denied: {} {}", req.getMethod(), req.getRequestURI());
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(ApiError.of("FORBIDDEN", "Access denied"));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiError> handleGeneral(Exception ex, HttpServletRequest req) {
        // Full details logged internally; safe message to client
        log.error("Unhandled exception: {} {}", req.getMethod(), req.getRequestURI(), ex);
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ApiError.of("INTERNAL_ERROR", "An internal error occurred"));
    }
}
```

---

## Configuration Management

```yaml
# application.yml — NEVER include secrets
spring:
  datasource:
    url: ${DATABASE_URL}         # from environment variable
    username: ${DATABASE_USER}
    password: ${DATABASE_PASSWORD}
  
  jpa:
    hibernate:
      ddl-auto: validate          # never 'create-drop' or 'create' in production
    show-sql: false               # never true in production (leaks query structure)

app:
  security:
    jwt:
      secret: ${JWT_SECRET}       # from environment variable or secrets manager
      expiration-seconds: ${JWT_EXPIRATION_SECONDS:900}

# Actuator — limit what is exposed
management:
  endpoints:
    web:
      exposure:
        include: health,info
  endpoint:
    health:
      show-details: when-authorized
```

---

## Actuator Security

```java
// Secure actuator endpoints in production
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    @Order(1)  // higher priority than main security config
    public SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
                .anyRequest().hasRole("ADMIN")
            )
            .build();
    }
}
```

---

## Data Access Patterns

```java
// ✅ Spring Data JPA — parameterized queries automatically
public interface UserRepository extends JpaRepository<UserEntity, UUID> {
    Optional<UserEntity> findByEmailAndActiveTrue(String email);

    // ✅ JPQL — named parameters only
    @Query("SELECT u FROM UserEntity u WHERE u.tenantId = :tenantId AND u.active = true")
    List<UserEntity> findAllByTenant(@Param("tenantId") UUID tenantId);

    // ❌ NEVER use string concatenation in @Query
}

// ✅ Tenant isolation — always filter by tenantId
@Service
public class UserService {
    public List<User> listUsers(UUID tenantId) {
        return userRepository.findAllByTenant(tenantId)  // tenantId always applied
            .stream().map(userMapper::toUser).toList();
    }
}
```

---

## Logging

```java
// ✅ Good — structured logging with context; never log sensitive data
@Slf4j
@Service
public class AuthService {

    public AuthToken authenticate(String email, String password) {
        try {
            var user = userRepository.findByEmailAndActiveTrue(email)
                .orElseThrow(() -> new AuthenticationException("Invalid credentials"));

            if (!passwordEncoder.matches(password, user.getPasswordHash())) {
                log.warn("auth.login.failure email={}", email);  // log email for investigation
                throw new AuthenticationException("Invalid credentials");
            }

            log.info("auth.login.success userId={}", user.getId());
            // ❌ NEVER: log.info("Login: email={} password={}", email, password);
            
            return tokenService.generate(user);
        } catch (AuthenticationException e) {
            throw e;
        } catch (Exception e) {
            log.error("auth.login.error email={}", email, e);
            throw new ServiceException("Authentication failed", e);
        }
    }
}
```
