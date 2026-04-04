---
name: dotnet-tdd
description: "Test-driven development for .NET using xUnit, FluentAssertions, NSubstitute, WebApplicationFactory, Testcontainers, and Coverlet. Use when adding features, fixing bugs, or refactoring."
origin: ECC
---

# .NET TDD Workflow

TDD guidance for .NET services with 80%+ coverage (unit + integration).

## When to Use

- New features or endpoints
- Bug fixes or refactors
- Adding data access logic or security rules

## Workflow

1) Write tests first (they should fail)
2) Implement minimal code to pass
3) Refactor with tests green
4) Enforce coverage (Coverlet)

## Unit Tests (xUnit + NSubstitute)

```csharp
public sealed class OrderServiceTests
{
    private readonly IOrderRepository _repository = Substitute.For<IOrderRepository>();
    private readonly ILogger<OrderService> _logger = Substitute.For<ILogger<OrderService>>();
    private readonly OrderService _sut;

    public OrderServiceTests()
    {
        _sut = new OrderService(_repository, _logger);
    }

    [Fact]
    public async Task PlaceOrderAsync_ReturnsSuccess_WhenRequestIsValid()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = "cust-123",
            Items = [new OrderItem("SKU-001", 2, 29.99m)]
        };

        // Act
        var result = await _sut.PlaceOrderAsync(request, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
        result.Value.Should().NotBeNull();
        result.Value!.CustomerId.Should().Be("cust-123");
    }

    [Fact]
    public async Task PlaceOrderAsync_ReturnsFailure_WhenNoItems()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = "cust-123",
            Items = []
        };

        // Act
        var result = await _sut.PlaceOrderAsync(request, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Contain("at least one item");
    }
}
```

Patterns:
- Arrange-Act-Assert
- One `_sut` (system under test) per test class
- Prefer NSubstitute over Moq for clean syntax
- Use `[Theory]` with `[InlineData]` for parameterized tests

## Parameterized Tests

```csharp
[Theory]
[InlineData("", false)]
[InlineData("a", false)]
[InlineData("ab@c.d", false)]
[InlineData("user@example.com", true)]
[InlineData("user+tag@example.co.uk", true)]
public void IsValidEmail_ReturnsExpected(string email, bool expected)
{
    EmailValidator.IsValid(email).Should().Be(expected);
}
```

## Web Layer Tests (WebApplicationFactory)

```csharp
public sealed class OrderApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrderApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                services.RemoveAll<DbContextOptions<AppDbContext>>();
                services.AddDbContext<AppDbContext>(options =>
                    options.UseInMemoryDatabase("TestDb"));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task GetOrder_Returns404_WhenNotFound()
    {
        var response = await _client.GetAsync($"/api/orders/{Guid.NewGuid()}");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }

    [Fact]
    public async Task CreateOrder_Returns201_WithValidRequest()
    {
        var request = new CreateOrderRequest
        {
            CustomerId = "cust-1",
            Items = [new("SKU-001", 1, 19.99m)]
        };

        var response = await _client.PostAsJsonAsync("/api/orders", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
        response.Headers.Location.Should().NotBeNull();
    }
}
```

## Integration Tests (Testcontainers)

```csharp
public sealed class PostgresOrderRepositoryTests : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    private AppDbContext _db = null!;

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(_postgres.GetConnectionString())
            .Options;
        _db = new AppDbContext(options);
        await _db.Database.MigrateAsync();
    }

    public async Task DisposeAsync()
    {
        await _db.DisposeAsync();
        await _postgres.DisposeAsync();
    }

    [Fact]
    public async Task AddAsync_PersistsOrder()
    {
        var repo = new SqlOrderRepository(_db);
        var order = Order.Create("cust-1", [new OrderItem("SKU-001", 2, 10m)]);

        await repo.AddAsync(order, CancellationToken.None);

        var found = await repo.FindByIdAsync(order.Id, CancellationToken.None);
        found.Should().NotBeNull();
        found!.Items.Should().HaveCount(1);
    }
}
```

## Coverage (Coverlet)

Add to test project:
```xml
<PackageReference Include="coverlet.collector" Version="6.*">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
</PackageReference>
```

Run with coverage:
```bash
dotnet test --collect:"XPlat Code Coverage"

# Generate HTML report
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coveragereport"
```

## Assertions (FluentAssertions)

```csharp
// Objects
result.Should().NotBeNull();
result.Should().BeEquivalentTo(expected);

// Collections
orders.Should().HaveCount(3);
orders.Should().ContainSingle(o => o.Status == OrderStatus.Active);

// Exceptions
var act = () => service.ProcessAsync(null!, CancellationToken.None);
await act.Should().ThrowAsync<ArgumentNullException>()
    .WithParameterName("request");

// Strings
message.Should().StartWith("Order").And.Contain("created");
```

## Test Data Builders

```csharp
public sealed class OrderBuilder
{
    private string _customerId = "cust-default";
    private readonly List<OrderItem> _items = [new("SKU-001", 1, 10m)];

    public OrderBuilder WithCustomer(string customerId)
    {
        _customerId = customerId;
        return this;
    }

    public OrderBuilder WithItem(string sku, int quantity, decimal price)
    {
        _items.Add(new OrderItem(sku, quantity, price));
        return this;
    }

    public Order Build() => Order.Create(_customerId, _items);
}
```

## Test Organization

```
tests/
  MyApp.UnitTests/
    Services/
      OrderServiceTests.cs
      PaymentServiceTests.cs
    Validators/
      EmailValidatorTests.cs
  MyApp.IntegrationTests/
    Api/
      OrderApiTests.cs
    Repositories/
      OrderRepositoryTests.cs
  MyApp.TestHelpers/
    Builders/
      OrderBuilder.cs
    Fixtures/
      DatabaseFixture.cs
```

## CI Commands

```bash
dotnet test --configuration Release --collect:"XPlat Code Coverage"
dotnet test --filter "Category=Unit"
dotnet test --filter "Category=Integration"
```

**Remember**: Keep tests fast, isolated, and deterministic. Test behavior, not implementation details. Write the test first — it drives the design.
