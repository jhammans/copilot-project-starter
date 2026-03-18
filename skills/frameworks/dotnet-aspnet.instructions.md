---
applyTo: "**/*.cs,**/Controllers/**,**/Program.cs,**/Startup.cs"
---

# ASP.NET Core Coding Standards

Apply these standards to all ASP.NET Core applications. Assumes .NET 8 with Minimal API or MVC Controllers.

---

## Project Setup

- Framework: **ASP.NET Core 8**
- Auth library: **Microsoft.AspNetCore.Authentication.JwtBearer** or **Duende IdentityServer**
- ORM: **Entity Framework Core 8** with code-first migrations
- Validation: **FluentValidation** + Data Annotations
- OpenAPI: **Swashbuckle** or **Scalar** (for .NET 9+)
- Logging: **Serilog** with structured JSON output

---

## Project Structure

```
src/
├── Api/                          # API project
│   ├── Program.cs                # App entry point and DI registration
│   ├── Controllers/
│   │   └── v1/
│   │       ├── UsersController.cs
│   │       └── DocumentsController.cs
│   ├── Middleware/
│   │   ├── ErrorHandlingMiddleware.cs
│   │   └── RequestLoggingMiddleware.cs
│   └── Filters/
│       └── ValidationFilter.cs
├── Application/                  # Business logic (no framework dependencies)
│   ├── Users/
│   │   ├── CreateUser/
│   │   │   ├── CreateUserCommand.cs
│   │   │   └── CreateUserHandler.cs
│   │   └── GetUser/
│   │       ├── GetUserQuery.cs
│   │       └── GetUserHandler.cs
│   └── Common/
│       └── Interfaces/
├── Domain/                       # Domain entities, value objects, domain events
├── Infrastructure/               # EF Core, external service implementations
│   ├── Persistence/
│   │   ├── AppDbContext.cs
│   │   └── Migrations/
│   └── Identity/
└── Tests/
    ├── Unit/
    └── Integration/
```

---

## Security Configuration

### JWT Authentication

```csharp
// Program.cs — register authentication
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Secret"]!)
            ),
            ClockSkew = TimeSpan.FromSeconds(30), // strict clock skew
        };
        
        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = ctx =>
            {
                // Log failure; return generic message
                var logger = ctx.HttpContext.RequestServices
                    .GetRequiredService<ILogger<Program>>();
                logger.LogWarning("JWT auth failed: {Error}", ctx.Exception.GetType().Name);
                return Task.CompletedTask;
            },
        };
    });
```

### Authorization Policies

```csharp
// Program.cs — register authorization policies
builder.Services.AddAuthorization(options =>
{
    // Permission-based policies
    options.AddPolicy("documents:read",
        p => p.RequireClaim("permission", "documents:read"));
    options.AddPolicy("documents:delete",
        p => p.RequireClaim("permission", "documents:delete"));
    options.AddPolicy("admin:access",
        p => p.RequireClaim("permission", "admin:access")
              .RequireClaim("amr", "mfa")); // require MFA claim for admin
});
```

### Controller Security

```csharp
[ApiController]
[Route("api/v1/[controller]")]
[Authorize]  // Require authentication for all actions in this controller
public class DocumentsController : ControllerBase
{
    private readonly IDocumentService _documentService;
    private readonly ILogger<DocumentsController> _logger;

    public DocumentsController(IDocumentService documentService, ILogger<DocumentsController> logger)
    {
        _documentService = documentService;
        _logger = logger;
    }

    [HttpGet]
    [Authorize(Policy = "documents:read")]
    public async Task<ActionResult<PagedResult<DocumentDto>>> List(
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 20,
        CancellationToken ct = default)
    {
        // Clamp page size to prevent DoS
        pageSize = Math.Clamp(pageSize, 1, 100);
        var userId = User.GetUserId(); // extension method on ClaimsPrincipal
        var result = await _documentService.ListAsync(userId, page, pageSize, ct);
        return Ok(result);
    }

    [HttpDelete("{id:guid}")]
    [Authorize(Policy = "documents:delete")]
    public async Task<IActionResult> Delete(Guid id, CancellationToken ct)
    {
        var userId = User.GetUserId();
        await _documentService.DeleteAsync(id, userId, ct);
        return NoContent();
    }
}
```

---

## Security Headers Middleware

```csharp
// Program.cs — configure security middleware
var app = builder.Build();

app.UseHsts(); // HSTS header
app.UseHttpsRedirection();

// Custom security headers
app.Use(async (context, next) =>
{
    context.Response.Headers["X-Content-Type-Options"] = "nosniff";
    context.Response.Headers["X-Frame-Options"] = "DENY";
    context.Response.Headers["Referrer-Policy"] = "strict-origin-when-cross-origin";
    context.Response.Headers["Permissions-Policy"] = "camera=(), microphone=()";
    context.Response.Headers["Content-Security-Policy"] =
        "default-src 'self'; frame-ancestors 'none';";
    await next();
});

app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

---

## Error Handling Middleware

```csharp
// Middleware/ErrorHandlingMiddleware.cs
public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ErrorHandlingMiddleware> _logger;

    public ErrorHandlingMiddleware(RequestDelegate next, ILogger<ErrorHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (NotFoundException ex)
        {
            context.Response.StatusCode = StatusCodes.Status404NotFound;
            await context.Response.WriteAsJsonAsync(new ApiError("NOT_FOUND", ex.Message));
        }
        catch (ForbiddenException)
        {
            context.Response.StatusCode = StatusCodes.Status403Forbidden;
            await context.Response.WriteAsJsonAsync(new ApiError("FORBIDDEN", "Access denied"));
        }
        catch (ValidationException ex)
        {
            context.Response.StatusCode = StatusCodes.Status400BadRequest;
            await context.Response.WriteAsJsonAsync(new ApiError("VALIDATION_ERROR", ex.Message));
        }
        catch (Exception ex)
        {
            // Full details in logs; generic message to client
            _logger.LogError(ex, "Unhandled exception for {Method} {Path}",
                context.Request.Method, context.Request.Path);
            
            context.Response.StatusCode = StatusCodes.Status500InternalServerError;
            await context.Response.WriteAsJsonAsync(new ApiError(
                "INTERNAL_ERROR",
                "An internal error occurred",
                context.TraceIdentifier
            ));
        }
    }
}
```

---

## Data Access — Entity Framework Core

```csharp
// Infrastructure/Persistence/AppDbContext.cs
public class AppDbContext : DbContext
{
    private readonly ICurrentUserService _currentUser;

    public AppDbContext(DbContextOptions<AppDbContext> options, ICurrentUserService currentUser)
        : base(options)
    {
        _currentUser = currentUser;
    }

    public DbSet<User> Users => Set<User>();
    public DbSet<Document> Documents => Set<Document>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        // ✅ Global query filters for multi-tenancy and soft deletes
        builder.Entity<Document>().HasQueryFilter(d =>
            d.TenantId == _currentUser.TenantId && !d.IsDeleted);
    }

    // ✅ Automatic audit fields
    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        foreach (var entry in ChangeTracker.Entries<IAuditableEntity>())
        {
            if (entry.State == EntityState.Added)
                entry.Entity.CreatedBy = _currentUser.UserId;
            if (entry.State is EntityState.Added or EntityState.Modified)
                entry.Entity.UpdatedAt = DateTimeOffset.UtcNow;
        }
        return base.SaveChangesAsync(ct);
    }
}
```

---

## Configuration (Never Hard-Code Secrets)

```csharp
// appsettings.json — structural config only; no secrets
{
  "Jwt": {
    "Issuer": "https://auth.example.com",
    "Audience": "https://api.example.com"
    // Secret is NOT here — it comes from environment or Key Vault
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}

// Program.cs — add Azure Key Vault in production
if (!builder.Environment.IsDevelopment())
{
    builder.Configuration.AddAzureKeyVault(
        new Uri($"https://{builder.Configuration["KeyVault:Name"]}.vault.azure.net/"),
        new DefaultAzureCredential()  // uses managed identity in production
    );
}
```

---

## Logging with Serilog

```csharp
// Program.cs — configure Serilog
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()
    .Enrich.WithCorrelationId()
    .WriteTo.Console(new JsonFormatter())
    .CreateLogger();

builder.Host.UseSerilog();

// ✅ Never log sensitive data
_logger.LogInformation(
    "User {UserId} authenticated from {IpAddress}",
    userId, ipAddress);

// ❌ Never
_logger.LogInformation("Login: email={Email} password={Password}", email, password);
```
