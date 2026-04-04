# .NET API — Project CLAUDE.md

> Real-world example for an ASP.NET Core Web API with PostgreSQL, EF Core, and Docker.
> Copy this to your project root and customize for your service.

## Project Overview

**Stack:** .NET 8+, ASP.NET Core, PostgreSQL, EF Core 8, Docker, xUnit + FluentAssertions

**Architecture:** Clean architecture with Domain, Application, Infrastructure, and API layers. Minimal APIs or controllers with proper separation of concerns.

## Critical Rules

### C# Conventions

- Enable nullable reference types project-wide (`<Nullable>enable</Nullable>`)
- Use `sealed` on all classes not designed for inheritance
- Use `record` or `record struct` for immutable value-like models
- Depend on interfaces at service boundaries — inject via constructor DI
- Async all the way — never block with `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()`
- Pass `CancellationToken` through every async method

### Database

- All data access through EF Core with Fluent API configuration
- Migrations in `Migrations/` — never alter the database manually
- Use `AsNoTracking()` for all read-only queries
- Use `Include()` or projections to prevent N+1 — never rely on lazy loading
- All queries must use parameterized inputs — never string interpolation in raw SQL

### Error Handling

- Return `Result<T>` from service methods for expected failures — throw for unexpected
- Use `IExceptionHandler` or middleware for global exception handling
- Define domain-specific exceptions in the Domain layer
- Map domain errors to HTTP status codes in the API layer
- Never expose stack traces, SQL text, or file paths in responses

```csharp
// Domain layer — specific exceptions
public sealed class OrderNotFoundException : Exception
{
    public OrderNotFoundException(Guid orderId)
        : base($"Order {orderId} was not found") { }
}

// API layer — global exception handler
app.UseExceptionHandler(error =>
{
    error.Run(async context =>
    {
        var exception = context.Features.Get<IExceptionHandlerFeature>()?.Error;
        var (statusCode, message) = exception switch
        {
            OrderNotFoundException => (404, "Order not found"),
            ArgumentException e => (400, e.Message),
            _ => (500, "An unexpected error occurred")
        };
        context.Response.StatusCode = statusCode;
        await context.Response.WriteAsJsonAsync(new { error = message });
    });
});
```

### Code Style

- No emojis in code or comments
- PascalCase for public members, `_camelCase` for private fields
- Keep methods under 50 lines — extract helper methods
- One primary type per file
- Use structured logging with typed parameters — no string interpolation in log calls
- Prefer `nameof()` over magic strings

## File Structure

```
src/
  MyApp.Api/
    Program.cs                    # Host configuration, DI, middleware pipeline
    Endpoints/                    # Minimal API endpoint groups
      OrderEndpoints.cs
      UserEndpoints.cs
    Middleware/
      RequestTimingMiddleware.cs
    appsettings.json
    appsettings.Development.json
  MyApp.Application/
    Services/
      OrderService.cs
      IOrderService.cs
    DTOs/
      CreateOrderRequest.cs
      OrderDto.cs
    Validators/
      CreateOrderValidator.cs
  MyApp.Domain/
    Entities/
      Order.cs
      OrderItem.cs
      User.cs
    Interfaces/
      IOrderRepository.cs
      IUserRepository.cs
    Exceptions/
      OrderNotFoundException.cs
    ValueObjects/
      Money.cs
  MyApp.Infrastructure/
    Persistence/
      AppDbContext.cs
      Configurations/
        OrderConfiguration.cs
        UserConfiguration.cs
      Repositories/
        SqlOrderRepository.cs
        SqlUserRepository.cs
    Migrations/
      20240101000000_InitialCreate.cs
    Services/
      SmtpEmailSender.cs
tests/
  MyApp.UnitTests/
    Services/
      OrderServiceTests.cs
    Validators/
      CreateOrderValidatorTests.cs
  MyApp.IntegrationTests/
    Api/
      OrderEndpointTests.cs
    Repositories/
      OrderRepositoryTests.cs
  MyApp.TestHelpers/
    Builders/
      OrderBuilder.cs
    Fixtures/
      DatabaseFixture.cs
docker-compose.yml
Dockerfile
```

## Key Patterns

### Entity with Encapsulated State

```csharp
public sealed class Order
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public required string CustomerId { get; init; }
    public OrderStatus Status { get; private set; } = OrderStatus.Pending;
    public decimal Total { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; } = DateTimeOffset.UtcNow;

    private readonly List<OrderItem> _items = [];
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    public static Order Create(string customerId, IEnumerable<OrderItem> items)
    {
        var order = new Order { CustomerId = customerId };
        foreach (var item in items)
            order.AddItem(item);
        return order;
    }

    public void AddItem(OrderItem item)
    {
        _items.Add(item);
        Total = _items.Sum(i => i.Quantity * i.Price);
    }

    public void Ship()
    {
        if (Status != OrderStatus.Pending)
            throw new InvalidOperationException($"Cannot ship order in {Status} status");
        Status = OrderStatus.Shipped;
    }
}
```

### Repository Interface

```csharp
public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(Guid id, CancellationToken cancellationToken);
    Task<IReadOnlyList<Order>> FindByCustomerAsync(string customerId, CancellationToken cancellationToken);
    Task AddAsync(Order order, CancellationToken cancellationToken);
    Task SaveChangesAsync(CancellationToken cancellationToken);
}
```

### Service with Dependency Injection

```csharp
public sealed class OrderService : IOrderService
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;

    public OrderService(IOrderRepository repository, ILogger<OrderService> logger)
    {
        _repository = repository;
        _logger = logger;
    }

    public async Task<Result<OrderDto>> CreateAsync(
        CreateOrderRequest request,
        CancellationToken cancellationToken)
    {
        if (request.Items.Count == 0)
            return Result<OrderDto>.Failure("Order must contain at least one item");

        var order = Order.Create(request.CustomerId, request.Items.Select(i =>
            new OrderItem(i.Sku, i.Quantity, i.Price)));

        await _repository.AddAsync(order, cancellationToken);

        _logger.LogInformation("Created order {OrderId} for customer {CustomerId}",
            order.Id, order.CustomerId);

        return Result<OrderDto>.Success(OrderDto.From(order));
    }
}
```

### Minimal API Endpoints

```csharp
public static class OrderEndpoints
{
    public static void MapOrderEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/api/orders")
            .RequireAuthorization()
            .WithTags("Orders");

        group.MapGet("/{id:guid}", GetOrder);
        group.MapPost("/", CreateOrder);
    }

    private static async Task<IResult> GetOrder(
        Guid id,
        IOrderService service,
        CancellationToken cancellationToken)
    {
        var order = await service.GetByIdAsync(id, cancellationToken);
        return order is not null
            ? TypedResults.Ok(order)
            : TypedResults.NotFound();
    }

    private static async Task<IResult> CreateOrder(
        CreateOrderRequest request,
        IOrderService service,
        CancellationToken cancellationToken)
    {
        var result = await service.CreateAsync(request, cancellationToken);
        return result.IsSuccess
            ? TypedResults.Created($"/api/orders/{result.Value!.Id}", result.Value)
            : TypedResults.BadRequest(new { error = result.Error });
    }
}
```

### xUnit Tests

```csharp
public sealed class OrderServiceTests
{
    private readonly IOrderRepository _repo = Substitute.For<IOrderRepository>();
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _sut = new OrderService(_repo, Substitute.For<ILogger<OrderService>>());
    }

    [Fact]
    public async Task CreateAsync_ValidRequest_ReturnsSuccess()
    {
        var request = new CreateOrderRequest
        {
            CustomerId = "cust-123",
            Items = [new("SKU-001", 2, 29.99m)]
        };

        var result = await _sut.CreateAsync(request, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        result.Value!.CustomerId.Should().Be("cust-123");
        await _repo.Received(1).AddAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task CreateAsync_EmptyItems_ReturnsFailure()
    {
        var request = new CreateOrderRequest { CustomerId = "cust-1", Items = [] };

        var result = await _sut.CreateAsync(request, CancellationToken.None);

        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Contain("at least one item");
    }
}
```

## Environment Variables

```env
# Required
ASPNETCORE_ENVIRONMENT=Development
DATABASE_URL=Host=localhost;Port=5432;Database=myapp;Username=postgres;Password=postgres

# Optional
CORS_ORIGINS=https://app.example.com
JWT_AUTHORITY=https://auth.example.com
JWT_AUDIENCE=myapp-api
SMTP_HOST=smtp.example.com
SMTP_PORT=587
```

## Development Commands

```bash
# Run locally
dotnet run --project src/MyApp.Api

# Run with hot reload
dotnet watch --project src/MyApp.Api

# Run tests
dotnet test

# Run tests with coverage
dotnet test --collect:"XPlat Code Coverage"

# EF Core migrations
dotnet ef migrations add MigrationName --project src/MyApp.Infrastructure --startup-project src/MyApp.Api
dotnet ef database update --project src/MyApp.Infrastructure --startup-project src/MyApp.Api

# Docker
docker compose up -d
docker compose down
```

## Docker Configuration

```dockerfile
# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY *.sln .
COPY src/MyApp.Api/*.csproj src/MyApp.Api/
COPY src/MyApp.Application/*.csproj src/MyApp.Application/
COPY src/MyApp.Domain/*.csproj src/MyApp.Domain/
COPY src/MyApp.Infrastructure/*.csproj src/MyApp.Infrastructure/
RUN dotnet restore

COPY . .
RUN dotnet publish src/MyApp.Api -c Release -o /app/publish --no-restore

# Runtime stage
FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/publish .
EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "MyApp.Api.dll"]
```

## CI Pipeline Checklist

1. `dotnet restore` — Restore packages
2. `dotnet build --no-restore` — Compile
3. `dotnet format --verify-no-changes` — Check formatting
4. `dotnet test --no-build --collect:"XPlat Code Coverage"` — Tests + coverage
5. `dotnet list package --vulnerable` — Security scan
6. `docker build -t myapp .` — Container build
7. Health check: `GET /health` returns 200
