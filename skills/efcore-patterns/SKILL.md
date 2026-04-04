---
name: efcore-patterns
description: "EF Core patterns for entity design, relationships, query optimization, transactions, auditing, migrations, pagination, and connection pooling in ASP.NET Core."
origin: ECC
---

# EF Core Patterns

Use for data modeling, repositories, and performance tuning in ASP.NET Core.

## When to Activate

- Designing EF Core entities and table mappings
- Defining relationships (one-to-many, many-to-many)
- Optimizing queries (N+1 prevention, projections, split queries)
- Configuring transactions, auditing, or soft deletes
- Setting up pagination, sorting, or custom repository methods
- Configuring connection pooling or interceptors

## Entity Design

```csharp
public sealed class Order
{
    public Guid Id { get; private set; } = Guid.NewGuid();
    public required string CustomerId { get; init; }
    public OrderStatus Status { get; private set; } = OrderStatus.Pending;
    public decimal Total { get; private set; }
    public DateTimeOffset CreatedAt { get; private set; } = DateTimeOffset.UtcNow;
    public DateTimeOffset? UpdatedAt { get; private set; }

    private readonly List<OrderItem> _items = [];
    public IReadOnlyList<OrderItem> Items => _items.AsReadOnly();

    public void AddItem(string sku, int quantity, decimal price)
    {
        _items.Add(new OrderItem(sku, quantity, price));
        Total = _items.Sum(i => i.Quantity * i.Price);
        UpdatedAt = DateTimeOffset.UtcNow;
    }

    public void Cancel()
    {
        if (Status == OrderStatus.Shipped)
            throw new InvalidOperationException("Cannot cancel a shipped order");
        Status = OrderStatus.Cancelled;
        UpdatedAt = DateTimeOffset.UtcNow;
    }
}
```

## Fluent Configuration

Prefer Fluent API over data annotations for complex mappings:

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");
        builder.HasKey(o => o.Id);

        builder.Property(o => o.CustomerId)
            .IsRequired()
            .HasMaxLength(100);

        builder.Property(o => o.Status)
            .HasConversion<string>()
            .HasMaxLength(20);

        builder.Property(o => o.Total)
            .HasPrecision(18, 2);

        builder.HasMany(o => o.Items)
            .WithOne()
            .HasForeignKey("OrderId")
            .OnDelete(DeleteBehavior.Cascade);

        builder.HasIndex(o => o.CustomerId);
        builder.HasIndex(o => o.Status);
        builder.HasIndex(o => o.CreatedAt);
    }
}

// In DbContext
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
}
```

## Relationships and N+1 Prevention

```csharp
// BAD: N+1 — lazy loading in a loop
foreach (var order in orders)
{
    var items = order.Items; // Triggers separate query per order
}

// GOOD: Eager loading with Include
var orders = await db.Orders
    .Include(o => o.Items)
    .Where(o => o.CustomerId == customerId)
    .ToListAsync(cancellationToken);

// GOOD: Projection to avoid loading full entities
var summaries = await db.Orders
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderSummaryDto
    {
        Id = o.Id,
        ItemCount = o.Items.Count,
        Total = o.Total,
        Status = o.Status
    })
    .ToListAsync(cancellationToken);

// GOOD: Split query for collections that produce cartesian explosion
var orders = await db.Orders
    .Include(o => o.Items)
    .AsSplitQuery()
    .ToListAsync(cancellationToken);
```

## Repository Pattern

```csharp
public interface IOrderRepository
{
    Task<Order?> FindByIdAsync(Guid id, CancellationToken cancellationToken);
    Task<IReadOnlyList<Order>> FindByCustomerAsync(string customerId, CancellationToken cancellationToken);
    Task AddAsync(Order order, CancellationToken cancellationToken);
    Task SaveChangesAsync(CancellationToken cancellationToken);
}

public sealed class SqlOrderRepository : IOrderRepository
{
    private readonly AppDbContext _db;

    public SqlOrderRepository(AppDbContext db) => _db = db;

    public async Task<Order?> FindByIdAsync(Guid id, CancellationToken cancellationToken)
    {
        return await _db.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.Id == id, cancellationToken);
    }

    public async Task<IReadOnlyList<Order>> FindByCustomerAsync(
        string customerId,
        CancellationToken cancellationToken)
    {
        return await _db.Orders
            .Where(o => o.CustomerId == customerId)
            .OrderByDescending(o => o.CreatedAt)
            .AsNoTracking()
            .ToListAsync(cancellationToken);
    }

    public async Task AddAsync(Order order, CancellationToken cancellationToken)
    {
        _db.Orders.Add(order);
        await _db.SaveChangesAsync(cancellationToken);
    }

    public Task SaveChangesAsync(CancellationToken cancellationToken)
        => _db.SaveChangesAsync(cancellationToken);
}
```

## AsNoTracking for Read Paths

```csharp
// GOOD: Read-only queries don't need change tracking
var orders = await db.Orders
    .AsNoTracking()
    .Where(o => o.Status == OrderStatus.Active)
    .ToListAsync(cancellationToken);

// For entire DbContext in read-heavy scenarios
builder.Services.AddDbContext<ReadOnlyDbContext>(options =>
    options.UseNpgsql(connectionString)
           .UseQueryTrackingBehavior(QueryTrackingBehavior.NoTracking));
```

## Transactions

```csharp
// Implicit: SaveChangesAsync wraps all changes in a transaction
await _db.SaveChangesAsync(cancellationToken);

// Explicit: For multi-step operations
await using var transaction = await _db.Database.BeginTransactionAsync(cancellationToken);
try
{
    _db.Orders.Add(order);
    await _db.SaveChangesAsync(cancellationToken);

    await _paymentService.ChargeAsync(order, cancellationToken);

    await transaction.CommitAsync(cancellationToken);
}
catch
{
    await transaction.RollbackAsync(cancellationToken);
    throw;
}
```

## Auditing

```csharp
public abstract class AuditableEntity
{
    public DateTimeOffset CreatedAt { get; set; }
    public DateTimeOffset? UpdatedAt { get; set; }
    public string? CreatedBy { get; set; }
    public string? UpdatedBy { get; set; }
}

// Override SaveChangesAsync in DbContext
public override async Task<int> SaveChangesAsync(CancellationToken cancellationToken = default)
{
    var now = DateTimeOffset.UtcNow;
    var user = _currentUserService.UserId;

    foreach (var entry in ChangeTracker.Entries<AuditableEntity>())
    {
        switch (entry.State)
        {
            case EntityState.Added:
                entry.Entity.CreatedAt = now;
                entry.Entity.CreatedBy = user;
                break;
            case EntityState.Modified:
                entry.Entity.UpdatedAt = now;
                entry.Entity.UpdatedBy = user;
                break;
        }
    }

    return await base.SaveChangesAsync(cancellationToken);
}
```

## Soft Delete with Global Query Filters

```csharp
public interface ISoftDeletable
{
    bool IsDeleted { get; set; }
    DateTimeOffset? DeletedAt { get; set; }
}

// In configuration
builder.HasQueryFilter(o => !o.IsDeleted);

// To include deleted records
db.Orders.IgnoreQueryFilters().Where(o => o.IsDeleted).ToListAsync(ct);
```

## Pagination

```csharp
public sealed record PagedResult<T>(
    IReadOnlyList<T> Items,
    int TotalCount,
    int Page,
    int PageSize)
{
    public int TotalPages => (int)Math.Ceiling(TotalCount / (double)PageSize);
    public bool HasNextPage => Page < TotalPages;
    public bool HasPreviousPage => Page > 1;
}

public async Task<PagedResult<OrderDto>> GetOrdersAsync(
    int page, int pageSize, CancellationToken cancellationToken)
{
    var query = _db.Orders.AsNoTracking().OrderByDescending(o => o.CreatedAt);

    var totalCount = await query.CountAsync(cancellationToken);
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(o => new OrderDto(o.Id, o.CustomerId, o.Status, o.Total))
        .ToListAsync(cancellationToken);

    return new PagedResult<OrderDto>(items, totalCount, page, pageSize);
}
```

## Indexing and Performance

- Add indexes for common filter columns (status, foreign keys, dates)
- Use composite indexes matching query patterns
- Prefer projections over loading full entity graphs
- Batch operations with `AddRange`, `UpdateRange`
- Configure `MaxBatchSize` for bulk inserts

```csharp
// In Fluent API
builder.HasIndex(o => new { o.Status, o.CreatedAt });
builder.HasIndex(o => o.CustomerId);

// Unique constraint
builder.HasIndex(o => o.Email).IsUnique();
```

## Connection Pooling

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseNpgsql(connectionString, npgsqlOptions =>
    {
        npgsqlOptions.CommandTimeout(30);
        npgsqlOptions.EnableRetryOnFailure(
            maxRetryCount: 3,
            maxRetryDelay: TimeSpan.FromSeconds(5),
            errorCodesToAdd: null);
    }));

// Or use DbContext pooling for high-throughput scenarios
builder.Services.AddDbContextPool<AppDbContext>(options =>
    options.UseNpgsql(connectionString), poolSize: 128);
```

## Migrations

- Use EF Core migrations; never modify the database manually in production
- Keep migrations additive and idempotent where possible
- Review generated SQL before applying to production

```bash
# Add a migration
dotnet ef migrations add AddOrderIndex

# Generate SQL script for review
dotnet ef migrations script --idempotent -o migration.sql

# Apply migrations
dotnet ef database update

# Revert last migration (development only)
dotnet ef migrations remove
```

## Interceptors

```csharp
public sealed class SlowQueryInterceptor : DbCommandInterceptor
{
    private readonly ILogger<SlowQueryInterceptor> _logger;
    private const int SlowQueryThresholdMs = 200;

    public SlowQueryInterceptor(ILogger<SlowQueryInterceptor> logger) => _logger = logger;

    public override DbDataReader ReaderExecuted(
        DbCommand command, CommandExecutedEventData eventData, DbDataReader result)
    {
        if (eventData.Duration.TotalMilliseconds > SlowQueryThresholdMs)
        {
            _logger.LogWarning("Slow query ({DurationMs}ms): {Sql}",
                eventData.Duration.TotalMilliseconds, command.CommandText);
        }
        return result;
    }
}

// Registration
builder.Services.AddDbContext<AppDbContext>((sp, options) =>
    options.UseNpgsql(connectionString)
           .AddInterceptors(sp.GetRequiredService<SlowQueryInterceptor>()));
```

## Testing Data Access

- Prefer Testcontainers with a real database over InMemory provider
- InMemory provider does not enforce constraints, foreign keys, or SQL behavior
- Assert query efficiency with logging: enable `Microsoft.EntityFrameworkCore.Database.Command` at `Information` level

**Remember**: Keep entities lean, queries intentional, and transactions short. Prevent N+1 with eager loading or projections, and index for your read/write patterns.
