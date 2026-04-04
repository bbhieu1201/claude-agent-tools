---
description: Enforce TDD workflow for .NET. Write xUnit tests first with FluentAssertions, then implement. Verify 80%+ coverage with Coverlet.
---

# .NET TDD Command

This command enforces test-driven development methodology for C# code using idiomatic .NET testing patterns.

## What This Command Does

1. **Define Types/Interfaces**: Scaffold signatures and contracts first
2. **Write Tests**: Create unit tests with xUnit and FluentAssertions (RED)
3. **Run Tests**: Verify tests fail for the right reason
4. **Implement Code**: Write minimal code to pass (GREEN)
5. **Refactor**: Improve while keeping tests green
6. **Check Coverage**: Ensure 80%+ coverage

## When to Use

Use `/dotnet-test` when:
- Implementing new C# services or endpoints
- Adding test coverage to existing code
- Fixing bugs (write failing test first)
- Building critical business logic
- Learning TDD workflow in .NET

## TDD Cycle

```
RED     → Write failing xUnit test with FluentAssertions
GREEN   → Implement minimal code to pass
REFACTOR → Improve code, tests stay green
REPEAT  → Next test case
```

## Example Session

````text
User: /dotnet-test I need a service to validate email addresses

Agent:
# TDD Session: Email Validator

## Step 1: Define Interface

```csharp
// Validators/IEmailValidator.cs
namespace MyApp.Validators;

public interface IEmailValidator
{
    bool IsValid(string email);
}
```

## Step 2: Write Tests (RED)

```csharp
// Tests/Validators/EmailValidatorTests.cs
namespace MyApp.UnitTests.Validators;

public sealed class EmailValidatorTests
{
    private readonly IEmailValidator _sut = new EmailValidator();

    [Theory]
    [InlineData("user@example.com", true)]
    [InlineData("user+tag@example.co.uk", true)]
    [InlineData("first.last@example.com", true)]
    [InlineData("", false)]
    [InlineData("userexample.com", false)]
    [InlineData("user@", false)]
    [InlineData("@example.com", false)]
    [InlineData("user@@example.com", false)]
    [InlineData("user @example.com", false)]
    public void IsValid_ReturnsExpected(string email, bool expected)
    {
        _sut.IsValid(email).Should().Be(expected);
    }
}
```

## Step 3: Run Tests — Confirm Failure

```bash
$ dotnet test --filter "EmailValidatorTests"
Failed: 9 tests
```
✅ Tests fail because `EmailValidator` is not implemented yet.

## Step 4: Implement (GREEN)

```csharp
// Validators/EmailValidator.cs
using System.Net.Mail;

namespace MyApp.Validators;

public sealed class EmailValidator : IEmailValidator
{
    public bool IsValid(string email)
    {
        if (string.IsNullOrWhiteSpace(email))
            return false;

        try
        {
            var addr = new MailAddress(email);
            return addr.Address == email
                && email.Contains('.')
                && !email.Contains(' ');
        }
        catch (FormatException)
        {
            return false;
        }
    }
}
```

## Step 5: Run Tests — Confirm Pass

```bash
$ dotnet test --filter "EmailValidatorTests"
Passed: 9 tests
```
✅ All tests pass.

## Step 6: Check Coverage

```bash
$ dotnet test --collect:"XPlat Code Coverage"
Line coverage: 92%
```
✅ Coverage exceeds 80% threshold.
````

## Test Patterns

### Unit Test with Mocking

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
    public async Task CreateAsync_ValidOrder_PersistsAndReturns()
    {
        // Arrange
        var request = new CreateOrderRequest
        {
            CustomerId = "cust-1",
            Items = [new("SKU-001", 2, 29.99m)]
        };

        // Act
        var result = await _sut.CreateAsync(request, CancellationToken.None);

        // Assert
        result.IsSuccess.Should().BeTrue();
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

### Integration Test with WebApplicationFactory

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
                services.AddDbContext<AppDbContext>(o =>
                    o.UseInMemoryDatabase("TddTest"));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task CreateOrder_Returns201_WhenValid()
    {
        var request = new { CustomerId = "cust-1", Items = new[] { new { Sku = "SKU-001", Quantity = 1, Price = 19.99m } } };

        var response = await _client.PostAsJsonAsync("/api/orders", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
    }
}
```

## Coverage Commands

```bash
# Run with coverage
dotnet test --collect:"XPlat Code Coverage"

# Generate HTML report
dotnet tool install -g dotnet-reportgenerator-globaltool
reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coverage/report"

# Run specific test category
dotnet test --filter "Category=Unit"
dotnet test --filter "Category=Integration"
```

## Related Skills

- `skill: csharp-testing` — Detailed testing patterns with xUnit and FluentAssertions
- `skill: dotnet-tdd` — Full TDD workflow guide
- `skill: dotnet-patterns` — C# architecture patterns
