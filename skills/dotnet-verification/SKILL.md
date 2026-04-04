---
name: dotnet-verification
description: "Verification loop for .NET projects: build, format, analyzers, tests with coverage, security scans, and diff review before release or PR."
origin: ECC
---

# .NET Verification Loop

Run before PRs, after major changes, and pre-deploy.

## When to Activate

- Before opening a pull request for a .NET service
- After major refactoring or NuGet upgrades
- Pre-deployment verification for staging or production
- Running full build → format → test → security scan pipeline
- Validating test coverage meets thresholds

## Phase 1: Build

```bash
dotnet build --configuration Release --no-incremental
```

If build fails, stop and fix using `csharp-build-resolver` agent.

## Phase 2: Format and Analyzers

```bash
# Check formatting
dotnet format --verify-no-changes

# If using .editorconfig analyzers
dotnet format analyzers --verify-no-changes

# Apply fixes if needed
dotnet format
```

## Phase 3: Tests + Coverage

```bash
dotnet test --configuration Release --collect:"XPlat Code Coverage" --results-directory ./coverage

# Generate report
reportgenerator -reports:"coverage/**/coverage.cobertura.xml" -targetdir:"coverage/report"
```

Report:
- Total tests, passed/failed/skipped
- Coverage % (lines/branches)
- Verify 80%+ line coverage

### Unit Tests

Test service logic in isolation with mocked dependencies:

```csharp
public sealed class UserServiceTests
{
    private readonly IUserRepository _repository = Substitute.For<IUserRepository>();
    private readonly UserService _sut;

    public UserServiceTests()
    {
        _sut = new UserService(_repository, Substitute.For<ILogger<UserService>>());
    }

    [Fact]
    public async Task CreateAsync_ValidInput_ReturnsUser()
    {
        var request = new CreateUserRequest { Name = "Alice", Email = "alice@example.com" };

        var result = await _sut.CreateAsync(request, CancellationToken.None);

        result.IsSuccess.Should().BeTrue();
        result.Value!.Name.Should().Be("Alice");
        await _repository.Received(1).AddAsync(Arg.Any<User>(), Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task CreateAsync_DuplicateEmail_ReturnsFailure()
    {
        _repository.ExistsByEmailAsync("existing@example.com", Arg.Any<CancellationToken>())
            .Returns(true);
        var request = new CreateUserRequest { Name = "Alice", Email = "existing@example.com" };

        var result = await _sut.CreateAsync(request, CancellationToken.None);

        result.IsSuccess.Should().BeFalse();
        result.Error.Should().Contain("already exists");
    }
}
```

### Integration Tests with WebApplicationFactory

Test endpoints through the full HTTP pipeline:

```csharp
public sealed class UserApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UserApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                services.RemoveAll<DbContextOptions<AppDbContext>>();
                services.AddDbContext<AppDbContext>(o =>
                    o.UseInMemoryDatabase("VerificationTest"));
            });
        }).CreateClient();
    }

    [Fact]
    public async Task CreateUser_Returns201_WhenValid()
    {
        var request = new { Name = "Alice", Email = "alice@example.com" };

        var response = await _client.PostAsJsonAsync("/api/users", request);

        response.StatusCode.Should().Be(HttpStatusCode.Created);
    }

    [Fact]
    public async Task CreateUser_Returns400_WhenInvalid()
    {
        var request = new { Name = "", Email = "not-an-email" };

        var response = await _client.PostAsJsonAsync("/api/users", request);

        response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
    }
}
```

### Integration Tests with Testcontainers

Test against real infrastructure:

```csharp
public sealed class UserRepositoryTests : IAsyncLifetime
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
    public async Task FindByEmailAsync_ExistingUser_ReturnsUser()
    {
        _db.Users.Add(new User { Name = "Alice", Email = "alice@example.com" });
        await _db.SaveChangesAsync();

        var found = await _db.Users
            .FirstOrDefaultAsync(u => u.Email == "alice@example.com");

        found.Should().NotBeNull();
        found!.Name.Should().Be("Alice");
    }
}
```

## Phase 4: Security Scan

```bash
# Vulnerable NuGet packages
dotnet list package --vulnerable

# Outdated packages (may have security patches)
dotnet list package --outdated

# Secrets in source
grep -rn "password\s*=\s*\"" --include="*.cs" --include="*.json" src/
grep -rn "sk-\|api_key\|secret" --include="*.cs" --include="*.json" src/

# Console.WriteLine usage (should use ILogger)
grep -rn "Console\.Write" --include="*.cs" src/

# Exception details leaked in responses
grep -rn "ex\.Message\|ex\.StackTrace" --include="*.cs" src/
```

## Phase 5: Diff Review

```bash
git diff --stat
git diff
```

Check for:
- Unintended changes to `.csproj` or `Directory.Build.props`
- New warnings introduced
- Missing `CancellationToken` on new async methods
- `appsettings.json` changes that might leak credentials

## Verification Summary Format

```text
## Verification Results

| Phase | Status | Details |
|-------|--------|---------|
| Build | ✅ PASS | Release build clean |
| Format | ✅ PASS | No format issues |
| Tests | ✅ PASS | 142 passed, 0 failed |
| Coverage | ✅ PASS | 87% line coverage |
| Security | ⚠️ WARN | 1 outdated package |
| Diff | ✅ PASS | Changes look intentional |

**Verdict: READY TO MERGE** (address outdated package in follow-up)
```

## References

See `skill: csharp-testing` for detailed test patterns.
See `skill: dotnet-security` for security review checklists.
