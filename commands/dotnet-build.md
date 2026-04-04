---
description: Fix .NET build errors, analyzer warnings, and NuGet issues incrementally. Invokes the csharp-build-resolver agent for minimal, surgical fixes.
---

# .NET Build and Fix

This command invokes the **csharp-build-resolver** agent to incrementally fix .NET build errors with minimal changes.

## What This Command Does

1. **Run Diagnostics**: Execute `dotnet build`, `dotnet format --verify-no-changes`
2. **Parse Errors**: Group by file and sort by severity
3. **Fix Incrementally**: One error at a time
4. **Verify Each Fix**: Re-run build after each change
5. **Report Summary**: Show what was fixed and what remains

## When to Use

Use `/dotnet-build` when:
- `dotnet build` fails with errors
- `dotnet format` reports violations
- NuGet package restore fails
- After pulling changes that break the build
- After upgrading .NET SDK or NuGet packages

## Diagnostic Commands Run

```bash
# Primary build check
dotnet build

# NuGet restore
dotnet restore

# Format check
dotnet format --verify-no-changes

# Check for vulnerable packages
dotnet list package --vulnerable

# Check NuGet sources
dotnet nuget list source
```

## Example Session

````text
User: /dotnet-build

Agent:
# .NET Build Resolution

## Initial Diagnostics

```bash
$ dotnet build
src/Services/PaymentService.cs(25,15): error CS0246: The type or namespace name 'Stripe' could not be found
src/Controllers/OrderController.cs(42,9): error CS0029: Cannot implicitly convert type 'string' to 'int'
src/Models/Order.cs(18,2): error CS8618: Non-nullable property 'CustomerId' must contain a non-null value
```

Errors found: 3

## Fix 1: Missing NuGet Package

File: src/Services/PaymentService.cs:25
Error: CS0246 — type or namespace 'Stripe' could not be found
Cause: Missing NuGet package

```bash
$ dotnet add src/MyApp.csproj package Stripe.net --version 43.0.0
$ dotnet build
# 2 errors remaining
```

## Fix 2: Type Mismatch

File: src/Controllers/OrderController.cs:42
Error: CS0029 — Cannot implicitly convert type 'string' to 'int'

```csharp
// Changed
var count = Request.Query["count"];
// To
var countStr = Request.Query["count"].ToString();
if (!int.TryParse(countStr, out var count))
    return BadRequest("Invalid count parameter");
```

```bash
$ dotnet build
# 1 error remaining
```

## Fix 3: Non-nullable Property

File: src/Models/Order.cs:18
Error: CS8618 — Non-nullable property must contain a non-null value

```csharp
// Changed
public string CustomerId { get; set; }
// To
public required string CustomerId { get; init; }
```

```bash
$ dotnet build
Build succeeded.
```

## Final Verification

```bash
$ dotnet test --no-build
Passed: 47, Failed: 0, Skipped: 0
```

## Summary

Build Status: SUCCESS | Errors Fixed: 3 | Files Modified: PaymentService.cs, OrderController.cs, Order.cs
````

## Common Error Categories

| Error Code | Category | Quick Fix |
|------------|----------|-----------|
| CS0246 | Missing type/namespace | Add `using` or NuGet package |
| CS0029 | Type conversion | Add explicit cast or `TryParse` |
| CS8618 | Non-nullable uninitialized | Add `required`, `= default!`, or init |
| CS0103 | Undefined name | Add variable declaration or `using` |
| CS0535 | Missing interface member | Implement the missing method |
| NU1605 | Dependency downgrade | Pin NuGet version |
| NETSDK1045 | SDK mismatch | Update `global.json` or target framework |

## Related Skills

- `skill: dotnet-patterns` — C# architecture patterns
- `skill: csharp-coding-standards` — Naming and coding conventions
