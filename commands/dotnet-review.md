---
description: Comprehensive C# code review for .NET conventions, async patterns, security, nullable reference types, and performance. Invokes the csharp-reviewer agent.
---

# .NET Code Review

This command invokes the **csharp-reviewer** agent for comprehensive C#-specific code review.

## What This Command Does

1. **Identify C# Changes**: Find modified `.cs` files via `git diff`
2. **Run Build and Format**: Execute `dotnet build` and `dotnet format --verify-no-changes`
3. **Security Scan**: Check for SQL injection, insecure deserialization, hardcoded secrets
4. **Async Review**: Analyze async/await patterns, CancellationToken usage, sync-over-async
5. **Nullable Safety**: Verify nullable reference type annotations and null checks
6. **Generate Report**: Categorize issues by severity

## When to Use

Use `/dotnet-review` when:
- After writing or modifying C# code
- Before committing .NET changes
- Reviewing pull requests with C# code
- Onboarding to a new .NET codebase
- Learning idiomatic C# patterns

## Review Categories

### CRITICAL (Must Fix)
- SQL/Command injection vulnerabilities
- Insecure deserialization (`BinaryFormatter`, `TypeNameHandling.All`)
- Hardcoded secrets (API keys, connection strings)
- Empty catch blocks swallowing exceptions
- Blocking async (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`)
- Missing `using`/`await using` for `IDisposable`/`IAsyncDisposable`

### HIGH (Should Fix)
- Missing `CancellationToken` on public async APIs
- `async void` methods (except event handlers)
- Nullable reference type warnings suppressed with `!`
- Business logic in controllers (should be in service layer)
- Entity exposed in API response (should use DTO)
- Mutable shared state in singleton services

### MEDIUM (Consider)
- Non-idiomatic naming conventions
- String concatenation in loops (use `StringBuilder`)
- N+1 queries in EF Core
- Missing `AsNoTracking()` for read-only queries
- Missing `sealed` on non-inherited classes
- `IEnumerable` multiple enumeration

## Automated Checks Run

```bash
# Build check
dotnet build

# Format verification
dotnet format --verify-no-changes

# Run tests
dotnet test --no-build

# Coverage
dotnet test --collect:"XPlat Code Coverage"
```

## Example Usage

````text
User: /dotnet-review

Agent:
# C# Code Review Report

## Files Reviewed
- src/Services/OrderService.cs (modified)
- src/Controllers/OrderController.cs (modified)

## Build & Format
✓ dotnet build: Success
✓ dotnet format: No issues

## Issues Found

[CRITICAL] Blocking Async Call
File: src/Services/OrderService.cs:45
Issue: `.Result` used on async method — deadlock risk
```csharp
// BAD
var order = _repository.FindByIdAsync(id, ct).Result;

// FIX
var order = await _repository.FindByIdAsync(id, ct);
```

[HIGH] Missing CancellationToken
File: src/Controllers/OrderController.cs:28
Issue: Async endpoint does not accept CancellationToken
```csharp
// BAD
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(Guid id)

// FIX
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(Guid id, CancellationToken cancellationToken)
```

[MEDIUM] Missing AsNoTracking
File: src/Services/OrderService.cs:62
Issue: Read-only query tracks entities unnecessarily
```csharp
// BAD
return await _db.Orders.Where(o => o.CustomerId == id).ToListAsync(ct);

// FIX
return await _db.Orders.AsNoTracking().Where(o => o.CustomerId == id).ToListAsync(ct);
```

## Summary
| Severity | Count |
|----------|-------|
| CRITICAL | 1     |
| HIGH     | 1     |
| MEDIUM   | 1     |

**Verdict: BLOCK** — Fix CRITICAL and HIGH issues before merge.
````

## Related Skills

- `skill: dotnet-patterns` — C# architecture patterns and conventions
- `skill: csharp-testing` — Testing patterns with xUnit and FluentAssertions
- `skill: dotnet-security` — Security review checklist for ASP.NET Core
- `skill: efcore-patterns` — EF Core query optimization and entity design
