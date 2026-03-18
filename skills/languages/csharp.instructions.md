---
applyTo: "**/*.cs"
---

# C# / .NET Coding Standards

Apply these standards to all C# code. Assumes .NET 8 or later.

---

## Toolchain

- Framework: **.NET 8 LTS** (minimum)
- Language version: **C# 12**
- Analyzer: **Roslyn analyzers** (`Microsoft.CodeAnalysis.NetAnalyzers` — enabled by default)
- Additional analyzers: **SonarAnalyzer.CSharp**, **Microsoft.AspNetCore.Mvc.Api.Analyzers**
- Security scanner: **dotnet list package --vulnerable** in CI

### `.editorconfig` — Required Settings

```editorconfig
root = true

[*.cs]
indent_style = space
indent_size = 4
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true

dotnet_sort_system_directives_first = true
dotnet_diagnostic.CA1062.severity = error    # Validate arguments of public methods
dotnet_diagnostic.CA2100.severity = error    # SQL injection check
dotnet_diagnostic.CA5350.severity = error    # No weak cryptographic algorithms
dotnet_diagnostic.CA5351.severity = error    # No broken cryptographic algorithms
```

---

## Naming Conventions

| Construct | Convention | Example |
|-----------|-----------|---------|
| Classes, records, interfaces | `PascalCase` | `UserService`, `IUserRepository` |
| Methods, properties | `PascalCase` | `GetUserById`, `IsActive` |
| Local variables & parameters | `camelCase` | `userId`, `isActive` |
| Private fields | `_camelCase` | `_userRepository` |
| Constants | `PascalCase` | `MaxRetries`, `DefaultTimeout` |
| Interface names | `I` prefix | `IUserRepository`, `IAuthService` |
| Async methods | `Async` suffix | `GetUserByIdAsync` |

---

## Modern C# Patterns

### Records for Immutable DTOs

```csharp
// ✅ Good — immutable request/response objects
public sealed record CreateUserRequest(
    [Required][EmailAddress][MaxLength(255)] string Email,
    [Required][MinLength(12)][MaxLength(128)] string Password,
    [Required][MaxLength(100)] string FirstName,
    [Required][MaxLength(100)] string LastName
);

public sealed record UserResponse(
    Guid Id,
    string Email,
    string FirstName,
    string LastName,
    DateTimeOffset CreatedAt
);
```

### Nullable Reference Types (Enabled by Default in .NET 8)

```csharp
// Must be enabled in .csproj
<Nullable>enable</Nullable>

// ✅ Good — explicit nullability
public async Task<User?> GetUserByIdAsync(Guid id, CancellationToken ct = default)
{
    return await _context.Users
        .AsNoTracking()
        .FirstOrDefaultAsync(u => u.Id == id && u.IsActive, ct);
}

// ❌ Bad — ignoring nullability warnings
User user = await GetUserByIdAsync(id)!; // null-forgiving without check
```

### Result Pattern for Error Handling

```csharp
public sealed class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }

    private Result(bool isSuccess, T? value, string? error) { ... }

    public static Result<T> Success(T value) => new(true, value, null);
    public static Result<T> Failure(string error) => new(false, default, error);
}
```

---

## Exception Handling

```csharp
// ✅ Good — custom exception types
public abstract class AppException : Exception
{
    protected AppException(string message) : base(message) { }
    protected AppException(string message, Exception inner) : base(message, inner) { }
}

public sealed class NotFoundException : AppException
{
    public string Resource { get; }
    public string Id { get; }

    public NotFoundException(string resource, string id)
        : base($"{resource} with id '{id}' not found")
    {
        Resource = resource;
        Id = id;
    }
}

// ✅ Good — specific catch; add context; log at the boundary
try
{
    var user = await _userRepository.GetByIdAsync(userId, ct)
        ?? throw new NotFoundException("User", userId.ToString());
    return user;
}
catch (NotFoundException)
{
    throw; // let it propagate — caller decides how to handle
}
catch (Exception ex) when (ex is not OperationCanceledException)
{
    _logger.LogError(ex, "Failed to retrieve user {UserId}", userId);
    throw new ServiceException($"Failed to retrieve user {userId}", ex);
}
```

---

## Security Patterns

### Input Validation with Data Annotations + FluentValidation

```csharp
// Data annotations for simple cases
public sealed record CreateDocumentRequest(
    [Required][MaxLength(255)] string Title,
    [Required] string Content
);

// FluentValidation for complex rules
public class CreateDocumentValidator : AbstractValidator<CreateDocumentRequest>
{
    public CreateDocumentValidator()
    {
        RuleFor(x => x.Title)
            .NotEmpty()
            .MaximumLength(255)
            .Matches(@"^[\w\s\-\.]+$").WithMessage("Title contains invalid characters.");

        RuleFor(x => x.Content)
            .NotEmpty()
            .MaximumLength(100_000);
    }
}
```

### Never Hard-Code Secrets

```csharp
// ✅ Good — use configuration; in production, use Azure Key Vault / AWS Secrets Manager
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection("Jwt"));

// ✅ appsettings.json references environment variables
// {
//   "Jwt": {
//     "Secret": "" // populated from env var JWT__Secret or Key Vault
//   }
// }

// ❌ Bad
var jwtSecret = "my-super-secret-key"; // NEVER hard-code
```

### SQL — Always Use Parameterized Queries or EF Core

```csharp
// ✅ Good — EF Core (parameterized automatically)
var user = await _context.Users
    .Where(u => u.Email == email && u.IsActive)
    .FirstOrDefaultAsync(ct);

// ✅ Good — Dapper with parameterized query
var user = await connection.QueryFirstOrDefaultAsync<User>(
    "SELECT * FROM Users WHERE Email = @Email AND IsActive = 1",
    new { Email = email }  // parameterized
);

// ❌ Bad — string interpolation in SQL
var query = $"SELECT * FROM Users WHERE Email = '{email}'"; // SQL injection
```

### Authorization in ASP.NET Core

```csharp
// ✅ Good — policy-based authorization 
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("documents:delete", policy =>
        policy.RequireClaim("permission", "documents:delete"));
});

[ApiController]
[Route("api/v1/documents")]
[Authorize]  // require authentication on all endpoints
public class DocumentsController : ControllerBase
{
    [HttpDelete("{id:guid}")]
    [Authorize(Policy = "documents:delete")]
    public async Task<IActionResult> Delete(Guid id, CancellationToken ct)
    {
        await _documentService.DeleteAsync(id, User.GetUserId(), ct);
        return NoContent();
    }
}
```

### Never Expose Stack Traces to Clients

```csharp
// ✅ Good — global exception handler
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var feature = context.Features.Get<IExceptionHandlerFeature>();
        var ex = feature?.Error;
        
        // Log full details internally
        var logger = context.RequestServices.GetRequiredService<ILogger<Program>>();
        logger.LogError(ex, "Unhandled exception for request {TraceId}", context.TraceIdentifier);
        
        // Return safe response to client
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await context.Response.WriteAsJsonAsync(new
        {
            error = "An internal error occurred.",
            traceId = context.TraceIdentifier  // safe to return; helps with support
        });
    });
});
```

---

## XML Documentation Comments

All public classes, interfaces, records, and methods must have XML documentation:

```csharp
/// <summary>
/// Retrieves an active user by their unique identifier.
/// </summary>
/// <param name="id">The user's unique identifier.</param>
/// <param name="ct">Cancellation token for async operation.</param>
/// <returns>
/// The matching <see cref="User"/> if found and active;
/// <see langword="null"/> otherwise.
/// </returns>
/// <exception cref="ServiceException">
/// Thrown when a database error occurs.
/// </exception>
public async Task<User?> GetUserByIdAsync(Guid id, CancellationToken ct = default)
{ ... }
```

---

## Testing Standards

- Framework: **xUnit** with **FluentAssertions** + **Moq** or **NSubstitute**
- Integration tests: **WebApplicationFactory** + **Testcontainers.MsSql** / **Testcontainers.PostgreSql**
- Coverage: minimum 80% line coverage; 100% on security paths

```csharp
public class UserServiceTests
{
    private readonly Mock<IUserRepository> _userRepository = new();
    private readonly UserService _sut;

    public UserServiceTests()
    {
        _sut = new UserService(_userRepository.Object);
    }

    [Fact]
    public async Task GetUserByIdAsync_WhenUserExists_ReturnsUser()
    {
        // Arrange
        var userId = Guid.NewGuid();
        var expectedUser = UserFixtures.ActiveUser(userId);
        _userRepository.Setup(r => r.GetByIdAsync(userId, default))
            .ReturnsAsync(expectedUser);

        // Act
        var result = await _sut.GetUserByIdAsync(userId);

        // Assert
        result.Should().NotBeNull()
              .And.BeEquivalentTo(expectedUser);
    }
}
```
