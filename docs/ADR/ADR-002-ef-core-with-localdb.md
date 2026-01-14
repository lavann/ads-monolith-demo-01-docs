# ADR-002: Use Entity Framework Core with SQL Server LocalDB

## Status
Accepted

## Context
The application requires persistent storage for products, inventory, carts, and orders. We needed to choose a data access strategy that provides good developer experience and flexibility for future migrations to cloud databases.

## Decision
We will use **Entity Framework Core (EF Core)** with **SQL Server LocalDB** for development and testing.

## Rationale

### Chosen: EF Core + SQL Server

**Entity Framework Core**
- **Code-first approach**: Models defined in C#, migrations manage schema evolution
- **LINQ queries**: Type-safe database queries with IntelliSense support
- **Change tracking**: Automatic detection of entity modifications
- **Relationship management**: Easy navigation properties between entities (Orders → OrderLines, Cart → CartLines)
- **Database agnostic**: Can swap provider (PostgreSQL, MySQL, SQLite) with configuration change

**SQL Server LocalDB**
- **Zero-config for developers**: Installs with Visual Studio, no separate server required
- **Connection string compatibility**: Easy migration to Azure SQL Database (same connection string pattern)
- **Full SQL Server features**: Indexes, constraints, transactions work identically to production SQL Server
- **Automatic startup**: On-demand execution mode, no Windows service

### Alternatives Considered

**Dapper (micro-ORM)**
- More control over SQL
- Better performance for read-heavy workloads
- Requires hand-written SQL and mapping code
- No automatic change tracking or migrations

**ADO.NET (raw SQL)**
- Maximum performance and control
- Much more boilerplate code
- Manual parameter binding and SQL injection risk
- Not suitable for rapid development

**NoSQL (MongoDB, Cosmos DB)**
- Better for document-oriented data
- Overkill for structured relational data (products have fixed schema)
- Order and cart relationships map naturally to SQL

## Consequences

### Positive
- **Fast development**: Scaffolding tools generate migrations automatically
- **Type safety**: Compile-time errors for model changes
- **Easy seeding**: `SeedAsync()` method populates test data on startup
- **Migration management**: `dotnet ef migrations` tracks schema changes
- **Cloud-ready**: Direct path to Azure SQL Database with minimal changes

### Negative
- **Performance overhead**: Change tracking and LINQ translation add latency compared to raw SQL
- **N+1 query problems**: Easy to accidentally trigger lazy loading (mitigated with `Include()`)
- **Complex queries**: LINQ limitations force `FromSqlRaw()` for advanced scenarios
- **Concurrency issues**: No row locking by default (see inventory race condition in LLD)
- **LocalDB limitations**: 
  - Single-user database (not shared across developers)
  - Not suitable for production (requires SQL Server/Azure SQL)
  - 10GB database size limit

### Observed Patterns in Codebase

**Design-Time Factory**
- `DesignTimeDbContextFactory` allows EF tools to create context at design time
- Hardcoded connection string: `Server=(localdb)\\MSSQLLocalDB;Database=RetailMonolith;Trusted_Connection=True;MultipleActiveResultSets=true`

**Automatic Migrations**
```csharp
// Program.cs lines 24-29
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync(); // Apply migrations on startup
    await AppDbContext.SeedAsync(db);  // Seed data
}
```
- **Risk**: If migration fails in production, app won't start
- **Benefit**: Simplifies local development (no manual migration step)

**Index Strategy**
- Unique indexes on `Product.Sku` and `InventoryItem.Sku`
- No foreign key relationships defined (loose coupling between Product and InventoryItem)
- No indexes on `CustomerId` (potential performance issue if multiple customers exist)

**Missing Features**
- No optimistic concurrency tokens (risk of lost updates)
- No row versioning for inventory (see race condition in LLD)
- No audit fields (CreatedBy, UpdatedBy, UpdatedAt) except `Order.CreatedUtc`
- No soft deletes (data deleted permanently)

## Compliance with Best Practices

### Good
✅ DbContext registered as Scoped (one per request)  
✅ Async operations throughout (`ToListAsync`, `SaveChangesAsync`)  
✅ Connection string externalized to `appsettings.json`  
✅ Migrations tracked in source control  

### Needs Improvement
⚠️ No explicit transactions in `CheckoutService`  
⚠️ No concurrency control on `InventoryItem.Quantity`  
⚠️ No connection resilience (retry policies)  
⚠️ No query result caching (every request hits database)  

## Notes
EF Core with LocalDB is appropriate for this demonstration monolith. For production:
- Use Azure SQL Database or full SQL Server instance
- Add Polly retry policies for transient fault handling
- Implement optimistic concurrency with `[Timestamp]` or `[ConcurrencyCheck]`
- Add explicit transactions in `CheckoutService` with `BeginTransactionAsync()`
- Consider read replicas for product catalog queries
- Add query logging and slow query monitoring
