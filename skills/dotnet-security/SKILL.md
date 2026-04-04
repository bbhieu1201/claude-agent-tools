---
name: dotnet-security
description: "ASP.NET Core security best practices: authentication, authorization, input validation, CSRF, secrets management, CORS, headers, and dependency scanning."
origin: ECC
---

# ASP.NET Core Security Review

Use when adding auth, handling input, creating endpoints, or dealing with secrets in .NET projects.

## When to Activate

- Adding authentication (JWT, OAuth2, cookie-based)
- Implementing authorization (policy-based, role-based, resource-based)
- Validating user input (data annotations, FluentValidation)
- Configuring CORS, CSRF, or security headers
- Managing secrets (user secrets, Azure Key Vault, environment variables)
- Adding rate limiting or brute-force protection
- Scanning NuGet dependencies for CVEs

## Authentication

- Prefer JWT Bearer tokens for APIs; cookie auth for server-rendered apps
- Use `httpOnly`, `Secure`, `SameSite=Strict` for cookie authentication
- Validate tokens via middleware, not manual parsing

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromSeconds(30)
        };
    });
```

## Authorization

- Use policy-based authorization over role checks
- Define policies in `Program.cs`; enforce with `[Authorize]` or `RequireAuthorization()`
- Deny by default; explicitly allow public endpoints

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("AdminOnly", policy => policy.RequireRole("Admin"))
    .AddPolicy("CanEditOrder", policy =>
        policy.Requirements.Add(new OrderOwnerRequirement()));

// Minimal API
app.MapDelete("/api/orders/{id}", DeleteOrder)
    .RequireAuthorization("CanEditOrder");

// Controller
[Authorize("AdminOnly")]
[HttpGet("users")]
public async Task<IActionResult> ListUsers() { ... }
```

## Input Validation

- Validate DTOs at the boundary using data annotations or FluentValidation
- Never trust client input; re-validate server-side
- Reject invalid model state before executing business logic

```csharp
// Data annotations
public sealed class CreateOrderRequest
{
    [Required, StringLength(100)]
    public required string CustomerId { get; init; }

    [Required, MinLength(1)]
    public required List<OrderItemDto> Items { get; init; }
}

// FluentValidation
public sealed class CreateOrderValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Items).NotEmpty();
        RuleForEach(x => x.Items).ChildRules(item =>
        {
            item.RuleFor(i => i.Sku).NotEmpty();
            item.RuleFor(i => i.Quantity).GreaterThan(0);
        });
    }
}
```

## SQL Injection Prevention

- Always use parameterized queries with EF Core, Dapper, or ADO.NET
- Never concatenate user input into SQL strings
- Validate sort/filter fields against an allow list

```csharp
// BAD: String interpolation in raw SQL
var sql = $"SELECT * FROM Orders WHERE CustomerId = '{customerId}'";

// GOOD: EF Core parameterized
var orders = await db.Orders
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(cancellationToken);

// GOOD: Dapper parameterized
const string sql = "SELECT * FROM Orders WHERE CustomerId = @customerId";
var orders = await connection.QueryAsync<Order>(sql, new { customerId });
```

## Secrets Management

- Never hardcode secrets in source code or `appsettings.json`
- Use User Secrets for local development: `dotnet user-secrets set "Key" "Value"`
- Use Azure Key Vault, AWS Secrets Manager, or environment variables in production
- Validate required configuration at startup

```csharp
// BAD: Hardcoded
const string apiKey = "sk-live-abc123";

// GOOD: Configuration with validation
var stripeKey = builder.Configuration["Stripe:SecretKey"]
    ?? throw new InvalidOperationException("Stripe:SecretKey is not configured");

// GOOD: Options pattern with validation
builder.Services.AddOptions<StripeOptions>()
    .BindConfiguration("Stripe")
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

## CSRF Protection

- For MVC/Razor apps, keep anti-forgery enabled; use `[ValidateAntiForgeryToken]`
- For stateless APIs with Bearer tokens, anti-forgery is typically not needed
- For Blazor Server, anti-forgery is built into the framework

```csharp
// Stateless API — disable anti-forgery, rely on Bearer auth
builder.Services.AddAntiforgery(options =>
{
    options.SuppressXFrameOptionsHeader = false;
});

// MVC — ensure forms include the token
[HttpPost]
[ValidateAntiForgeryToken]
public async Task<IActionResult> CreateOrder(CreateOrderRequest request) { ... }
```

## Security Headers

```csharp
app.Use(async (context, next) =>
{
    context.Response.Headers.Append("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Append("X-Frame-Options", "DENY");
    context.Response.Headers.Append("X-XSS-Protection", "0");
    context.Response.Headers.Append("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Append(
        "Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self'");
    await next();
});
```

## CORS Configuration

- Configure CORS at the middleware level, not per-controller
- Never use `AllowAnyOrigin()` in production
- Restrict allowed methods and headers

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", policy =>
    {
        policy.WithOrigins("https://app.example.com")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Authorization", "Content-Type")
              .AllowCredentials()
              .SetPreflightMaxAge(TimeSpan.FromHours(1));
    });
});

app.UseCors("Production");
```

## Rate Limiting (.NET 7+)

```csharp
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("api", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueLimit = 0;
    });
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();

app.MapGet("/api/orders", GetOrders)
    .RequireRateLimiting("api");
```

## Password Hashing

- Use `Microsoft.AspNetCore.Identity.PasswordHasher<T>` or BCrypt — never store plaintext
- Never implement custom hashing

```csharp
var hasher = new PasswordHasher<User>();
string hashed = hasher.HashPassword(user, request.Password);

// Verification
var result = hasher.VerifyHashedPassword(user, user.PasswordHash, request.Password);
if (result == PasswordVerificationResult.Failed)
    return Unauthorized();
```

## Error Handling

- Return safe, generic messages to clients
- Log detailed exceptions server-side with structured logging
- Never expose stack traces, SQL, or file paths in API responses

```csharp
app.UseExceptionHandler(error =>
{
    error.Run(async context =>
    {
        context.Response.StatusCode = StatusCodes.Status500InternalServerError;
        context.Response.ContentType = "application/json";
        await context.Response.WriteAsJsonAsync(new { error = "An unexpected error occurred" });
    });
});
```

## Dependency Scanning

```bash
# Check for vulnerable NuGet packages
dotnet list package --vulnerable

# Outdated packages (may have security fixes)
dotnet list package --outdated

# Use dotnet-retire for deeper scanning
dotnet tool install -g dotnet-retire
dotnet retire
```

## Common Security Findings

```bash
# Check for hardcoded secrets
grep -rn "password\s*=\s*\"" --include="*.cs" --include="*.json" src/
grep -rn "sk-\|api_key\|secret" --include="*.cs" --include="*.json" src/

# Check for Console.WriteLine (use ILogger instead)
grep -rn "Console\.Write" --include="*.cs" src/

# Check for raw exception messages in responses
grep -rn "ex\.Message\|ex\.StackTrace" --include="*.cs" src/

# Check for wildcard CORS
grep -rn "AllowAnyOrigin" --include="*.cs" src/
```

## References

See `skill: security-review` for broader application security checklists.
See `skill: dotnet-patterns` for general .NET architecture patterns.
