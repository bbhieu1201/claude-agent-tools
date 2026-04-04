---
name: csharp-coding-standards
description: "C# coding standards for .NET services: naming, immutability, nullable reference types, async patterns, records, exceptions, and project layout."
origin: ECC
---

# C# Coding Standards

Standards for readable, maintainable C# (.NET 8+) code in ASP.NET Core services.

## When to Activate

- Writing or reviewing C# code in .NET projects
- Enforcing naming, immutability, or async conventions
- Working with records, nullable reference types, or pattern matching
- Reviewing LINQ usage, exception handling, or DI patterns
- Structuring projects and solution layout

## Core Principles

- Prefer clarity over cleverness
- Immutable by default; minimize shared mutable state
- Fail fast with meaningful exceptions
- Consistent naming and project structure
- Enable nullable reference types project-wide

## Naming

```csharp
// PASS: Types: PascalCase
public sealed class OrderService { }
public sealed record Money(decimal Amount, string Currency);
public interface IOrderRepository { }

// PASS: Methods and properties: PascalCase
public Task<Order?> FindByIdAsync(Guid id, CancellationToken cancellationToken);
public string CustomerName { get; init; }

// PASS: Private fields: _camelCase
private readonly IOrderRepository _repository;
private readonly ILogger<OrderService> _logger;

// PASS: Parameters and locals: camelCase
public async Task ProcessAsync(Guid orderId, CancellationToken cancellationToken)
{
    var order = await _repository.FindByIdAsync(orderId, cancellationToken);
}

// PASS: Constants: PascalCase
private const int MaxPageSize = 100;
public const string DefaultCurrency = "USD";
```

## Immutability

```csharp
// PASS: Use records for value-like models
public sealed record OrderDto(Guid Id, string CustomerId, OrderStatus Status);
public sealed record struct Coordinate(double Latitude, double Longitude);

// PASS: Use init-only properties for DTOs
public sealed class CreateOrderRequest
{
    public required string CustomerId { get; init; }
    public required IReadOnlyList<OrderItemDto> Items { get; init; }
}

// PASS: Immutable collections for shared state
public IReadOnlyList<string> Tags { get; init; } = [];
public IReadOnlyDictionary<string, int> Counts { get; init; }
    = new Dictionary<string, int>();
```

## Nullable Reference Types

```csharp
// PASS: Enable project-wide
// In .csproj: <Nullable>enable</Nullable>

// PASS: Explicit nullability
public async Task<User?> FindByEmailAsync(string email, CancellationToken ct);

// PASS: Null checks with pattern matching
if (user is null)
    return Result<Order>.Failure("User not found");

// PASS: Null-forgiving only when provably safe
var name = user!.Name; // Only after null check guarantees non-null

// FAIL: Suppressing nullable warnings broadly
#pragma warning disable CS8618
```

## Async Patterns

```csharp
// PASS: Async all the way with CancellationToken
public async Task<OrderSummary> GetSummaryAsync(
    Guid orderId,
    CancellationToken cancellationToken)
{
    var order = await _repository.FindByIdAsync(orderId, cancellationToken)
        ?? throw new NotFoundException($"Order {orderId} not found");

    return OrderSummary.From(order);
}

// PASS: Return Task directly when no await needed
public Task<int> CountAsync(CancellationToken cancellationToken)
    => _repository.CountAsync(cancellationToken);

// FAIL: Blocking on async
var order = _repository.FindByIdAsync(id, ct).Result; // Deadlock risk!

// FAIL: async void (except event handlers)
public async void ProcessOrder(Order order) { } // Unobservable exceptions
```

## Pattern Matching

```csharp
// PASS: Use pattern matching for type checks
if (result is SuccessResult<Order> success)
    return Ok(success.Value);

// PASS: Switch expressions for exhaustive matching
var statusCode = result switch
{
    { IsSuccess: true } => StatusCodes.Status200OK,
    { Error: "not_found" } => StatusCodes.Status404NotFound,
    { Error: "validation" } => StatusCodes.Status400BadRequest,
    _ => StatusCodes.Status500InternalServerError
};

// PASS: Property pattern in LINQ
var activeOrders = orders.Where(o => o is { Status: OrderStatus.Active, Total: > 0 });
```

## LINQ Best Practices

```csharp
// PASS: Use method syntax for complex queries
var summary = orders
    .Where(o => o.Status == OrderStatus.Active)
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, Total = g.Sum(o => o.Total) })
    .OrderByDescending(x => x.Total)
    .Take(10)
    .ToList();

// PASS: Materialize when enumerating multiple times
var items = source.Where(x => x.IsValid).ToList();
var count = items.Count;
var first = items.FirstOrDefault();

// FAIL: Complex nested LINQ; prefer explicit loops for readability
```

## Exceptions

- Use specific exceptions for domain errors
- Throw `ArgumentNullException`, `ArgumentOutOfRangeException` for input validation
- Use `InvalidOperationException` for illegal state
- Create domain-specific exceptions when needed

```csharp
// Guard clauses
ArgumentNullException.ThrowIfNull(request);
ArgumentOutOfRangeException.ThrowIfNegativeOrZero(request.Amount);

// Domain exceptions
public sealed class OrderNotFoundException : Exception
{
    public OrderNotFoundException(Guid orderId)
        : base($"Order {orderId} was not found") { }
}

// Result pattern for expected failures
public sealed record Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }

    public static Result<T> Success(T value) => new() { IsSuccess = true, Value = value };
    public static Result<T> Failure(string error) => new() { Error = error };
}
```

## Dependency Injection

```csharp
// PASS: Constructor injection with primary constructors (.NET 8+)
public sealed class OrderService(
    IOrderRepository repository,
    ILogger<OrderService> logger)
{
    public async Task<Order?> FindAsync(Guid id, CancellationToken ct)
    {
        logger.LogInformation("Finding order {OrderId}", id);
        return await repository.FindByIdAsync(id, ct);
    }
}

// PASS: Register lifetimes intentionally
builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddSingleton<ITimeProvider, SystemTimeProvider>();
builder.Services.AddTransient<IEmailSender, SmtpEmailSender>();
```

## Logging

```csharp
// PASS: Structured logging with typed parameters
logger.LogInformation("Processing order {OrderId} for customer {CustomerId}",
    order.Id, order.CustomerId);

logger.LogError(ex, "Failed to process payment for order {OrderId}", orderId);

// FAIL: String interpolation in log calls (defeats structured logging)
logger.LogInformation($"Processing order {order.Id}"); // Allocates even when not logging
```

## Project Structure

```
src/
  MyApp.Api/                  # ASP.NET Core host (Program.cs, controllers/endpoints)
  MyApp.Application/          # Business logic, services, DTOs, validators
  MyApp.Domain/               # Entities, value objects, interfaces, domain events
  MyApp.Infrastructure/       # EF Core, external services, email, storage
tests/
  MyApp.UnitTests/            # Service and domain logic tests
  MyApp.IntegrationTests/     # API and repository tests
  MyApp.TestHelpers/          # Shared builders, fixtures, fakes
```

## Sealed Classes

```csharp
// PASS: Seal non-inherited classes for clarity and performance
public sealed class OrderService { }
public sealed record OrderDto(Guid Id, string Name);

// Unsealed only when designed for inheritance
public abstract class BaseEntity { }
```

## Code Smells to Avoid

- Long parameter lists → use DTO/request objects
- Deep nesting → early returns / guard clauses
- Magic numbers/strings → named constants or `nameof()`
- Static mutable state → prefer DI scoping
- Empty catch blocks → log and rethrow or handle explicitly
- `dynamic` usage → prefer generics or explicit models

## Testing Expectations

- xUnit + FluentAssertions for fluent assertions
- NSubstitute or Moq for mocking dependencies
- Favor deterministic tests; no `Thread.Sleep`
- Name tests by behavior: `MethodName_Scenario_ExpectedResult`

**Remember**: Keep code intentional, typed, and observable. Optimize for maintainability over micro-optimizations unless proven necessary with benchmarks.
