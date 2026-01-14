# Runbook - Retail Monolith

## Overview
Operational guide for building, running, and troubleshooting the Retail Monolith application.

---

## Prerequisites

### Required Software
- **.NET 8 SDK** or later
  - Download from: https://dotnet.microsoft.com/download/dotnet/8.0
  - Verify: `dotnet --version` (should show 8.0.x or higher)

- **SQL Server LocalDB** (for development)
  - Included with Visual Studio 2022
  - Alternative: SQL Server Express or full SQL Server
  - Verify: `sqllocaldb info` (should list MSSQLLocalDB)

### Optional Tools
- **Visual Studio 2022** (Community or higher) - Full IDE experience
- **Visual Studio Code** with C# extension - Lightweight editor
- **SQL Server Management Studio (SSMS)** - Database inspection
- **dotnet-ef CLI** - Already included via project reference

---

## Initial Setup

### 1. Clone Repository
```bash
git clone https://github.com/lavann/ads-monolith-demo-01-docs.git
cd ads-monolith-demo-01-docs
```

### 2. Restore Dependencies
```bash
dotnet restore
```

**Expected Output:**
```
Determining projects to restore...
Restored RetailMonolith.csproj (in XXX ms).
```

### 3. Initialize Database
```bash
dotnet ef database update
```

**What This Does:**
- Creates `RetailMonolith` database in LocalDB
- Applies all migrations from `/Migrations` folder
- Does NOT seed data (seeding happens on first run)

**Expected Output:**
```
Build started...
Build succeeded.
Applying migration '20251019185248_Initial'.
Done.
```

### 4. Run Application
```bash
dotnet run
```

**What This Does:**
- Compiles application
- Applies any pending migrations
- Seeds 50 products with inventory (if database is empty)
- Starts Kestrel web server

**Expected Output:**
```
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: https://localhost:5001
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5000
```

### 5. Access Application
- **HTTPS**: https://localhost:5001
- **HTTP**: http://localhost:5000

**Available Routes:**
- `/` - Home page
- `/Products` - Product catalog
- `/Cart` - Shopping cart
- `/api/checkout` - Checkout API (POST)
- `/api/orders/{id}` - Order details API (GET)

---

## Common Commands

### Build
```bash
# Build project
dotnet build

# Build in Release mode
dotnet build -c Release

# Clean build artifacts
dotnet clean
```

### Run
```bash
# Run with Development environment
dotnet run

# Run with Production environment
dotnet run --environment Production

# Run on specific port
dotnet run --urls "https://localhost:7001;http://localhost:7000"

# Watch mode (auto-restart on file changes)
dotnet watch run
```

### Database Operations

#### View Migration History
```bash
dotnet ef migrations list
```

#### Create New Migration
```bash
# After modifying models in /Models folder
dotnet ef migrations add YourMigrationName

# Example
dotnet ef migrations add AddCustomerEmailField
```

#### Apply Migrations
```bash
# Apply all pending migrations
dotnet ef database update

# Rollback to specific migration
dotnet ef database update PreviousMigrationName

# Rollback all migrations (empty database)
dotnet ef database update 0
```

#### Reset Database
```bash
# Drop database
dotnet ef database drop -f

# Recreate and seed
dotnet ef database update
dotnet run
```

**Use Case:** When test data is corrupted or you want fresh seeded data.

#### Inspect Database
```bash
# List databases
sqllocaldb info

# Connect to database (if SSMS installed)
# Server name: (localdb)\MSSQLLocalDB
# Database: RetailMonolith
# Authentication: Windows Authentication
```

### Testing

**Note:** No test projects currently exist in solution.

```bash
# Run tests (when added)
dotnet test

# Run with coverage
dotnet test /p:CollectCoverage=true
```

---

## Configuration

### Connection String

**Default (LocalDB):**
```
Server=(localdb)\\MSSQLLocalDB;Database=RetailMonolith;Trusted_Connection=True;MultipleActiveResultSets=true
```

**Override via Environment Variable:**
```bash
# Linux/Mac
export ConnectionStrings__DefaultConnection="Server=your-server;Database=RetailMonolith;User Id=sa;Password=YourPassword;"
dotnet run

# Windows (Command Prompt)
set ConnectionStrings__DefaultConnection=Server=your-server;Database=RetailMonolith;User Id=sa;Password=YourPassword;
dotnet run

# Windows (PowerShell)
$env:ConnectionStrings__DefaultConnection="Server=your-server;Database=RetailMonolith;User Id=sa;Password=YourPassword;"
dotnet run
```

**Override via appsettings.json:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=your-server;Database=RetailMonolith;..."
  }
}
```

### Environment-Specific Settings

**appsettings.Development.json** (already exists, currently empty)
- Add development-only overrides here
- Example: verbose logging, test API keys

**appsettings.Production.json** (create if needed)
- Production-specific settings
- Should NOT be committed to source control if contains secrets

---

## Troubleshooting

### Error: "LocalDB is not installed"

**Symptom:**
```
System.Data.SqlClient.SqlException: A network-related or instance-specific error...
```

**Solution:**
1. Install SQL Server LocalDB:
   - Via Visual Studio Installer (select ".NET desktop development" workload)
   - Or download standalone: https://learn.microsoft.com/en-us/sql/database-engine/configure-windows/sql-server-express-localdb
2. Verify: `sqllocaldb info`
3. Start instance: `sqllocaldb start MSSQLLocalDB`

---

### Error: "Build failed" or Compilation Errors

**Symptom:**
```
CSC : error CS0006: Metadata file '...' could not be found
```

**Solution:**
```bash
dotnet clean
dotnet restore
dotnet build
```

---

### Error: Migration Already Applied

**Symptom:**
```
The migration '20251019185248_Initial' has already been applied to the database.
```

**Solution:**
- This is informational, not an error
- Migration already exists in database, no action needed
- If you want to reapply: `dotnet ef database drop -f` then `dotnet ef database update`

---

### Error: Port Already in Use

**Symptom:**
```
System.IO.IOException: Failed to bind to address https://127.0.0.1:5001
```

**Solution:**
1. Find process using port:
   ```bash
   # Windows
   netstat -ano | findstr :5001
   
   # Linux/Mac
   lsof -i :5001
   ```
2. Kill process or use different port:
   ```bash
   dotnet run --urls "https://localhost:6001;http://localhost:6000"
   ```

---

### Issue: No Products Displayed

**Symptom:** `/Products` page shows empty list

**Cause:** Database seeding failed or was skipped

**Solution:**
```bash
# Check if products exist in DB (via SSMS or code)
# If empty, reseed:
dotnet ef database drop -f
dotnet ef database update
dotnet run
```

**Verify Seeding:** On startup, console should show:
```
info: Microsoft.EntityFrameworkCore.Database.Command[20101]
      Executed DbCommand (Xms) [Parameters=[], CommandType='Text', CommandTimeout='30']
      INSERT INTO [Products] ...
```

---

### Issue: Cart Shared Between Users

**Symptom:** Adding item to cart in one browser shows in another browser

**Expected Behavior:** This is not a bug, it's by design (see ADR-004)

**Explanation:**
- All users share the same "guest" customer ID
- Cart is stored server-side in database, not per-session
- This is intentional for demonstration purposes

**Workaround for Testing:**
- Use different customer IDs (requires code change)
- Clear cart between tests: Delete `Carts` and `CartLines` from database

---

### Issue: Health Check Endpoint 404

**Symptom:** `GET /health` returns 404 Not Found

**Cause:** Health check service is registered but endpoint not mapped

**Evidence:**
```csharp
// Program.cs line 19
builder.Services.AddHealthChecks(); // ✅ Registered

// Missing:
// app.MapHealthChecks("/health"); // ❌ Not mapped
```

**Status:** Known issue (tech debt)

**Workaround:** Add mapping in `Program.cs` after line 47:
```csharp
app.MapHealthChecks("/health");
```

---

## Known Issues / Technical Debt

### 1. Hardcoded "guest" Customer
- **Impact:** All users share same cart and orders
- **Severity:** High (production blocker)
- **Reference:** ADR-004
- **Fix Required:** Implement ASP.NET Identity authentication

### 2. Mock Payment Gateway
- **Impact:** Payments always succeed, no real processing
- **Severity:** Critical (production blocker)
- **Reference:** ADR-003
- **Fix Required:** Integrate real payment provider (Stripe, PayPal)

### 3. Duplicate Cart Logic in ProductsIndexModel
- **Location:** `Pages/Products/Index.cshtml.cs` lines 32-48
- **Impact:** Confusing code, lines 32-40 are unused
- **Severity:** Low (code quality)
- **Fix Required:** Remove direct DbContext access, use only CartService

### 4. Inventory Race Condition
- **Location:** `CheckoutService.CheckoutAsync()` inventory decrement
- **Impact:** Concurrent checkouts can oversell products
- **Severity:** High (data integrity issue)
- **Reference:** LLD.md - Technical Hotspots #3
- **Fix Required:** Add row locking or optimistic concurrency

### 5. Health Check Not Exposed
- **Location:** `Program.cs` (missing `MapHealthChecks`)
- **Impact:** Cannot monitor application health
- **Severity:** Low (ops inconvenience)
- **Fix Required:** Add `app.MapHealthChecks("/health");`

### 6. No Transaction Management in Checkout
- **Location:** `CheckoutService.CheckoutAsync()`
- **Impact:** Failed SaveChanges leaves inconsistent state
- **Severity:** Medium (data integrity risk)
- **Fix Required:** Wrap in `using var transaction = await _db.Database.BeginTransactionAsync()`

### 7. Empty Page Models
- **Location:** `Checkout/IndexModel`, `Orders/IndexModel`
- **Impact:** No web UI for checkout or order history
- **Severity:** Low (API works, UI incomplete)
- **Fix Required:** Implement Razor Page views or remove placeholder pages

### 8. Payment Before Inventory Reserve
- **Location:** `CheckoutService` - inventory decremented before payment
- **Impact:** Failed payment leaves inventory decremented
- **Severity:** Medium (business logic issue)
- **Fix Required:** Charge payment first, or implement compensating transaction

### 9. No Connection Resilience
- **Location:** `Program.cs` DbContext registration
- **Impact:** Transient DB errors cause request failures
- **Severity:** Medium (production reliability)
- **Fix Required:** Add Polly retry policies to DbContext

### 10. Hardcoded Payment Token in API
- **Location:** `Program.cs` line 53
- **Impact:** API doesn't accept payment token from client
- **Severity:** High (API design flaw)
- **Fix Required:** Accept `CheckoutRequest` with token in body

### 11. Filename with Trailing Space
- **Location:** `Services/CheckoutService .cs` (note: space before .cs)
- **Impact:** Unusual filename may cause issues with some tools
- **Severity:** Low (code quality)
- **Fix Required:** Rename file to `CheckoutService.cs` (no space)

---

## Performance Considerations

### Database Queries
- **N+1 problem mitigated:** Use of `.Include()` for related entities
- **No caching:** Every request hits database (add Redis if needed)
- **No indexes on CustomerId:** May cause slow queries if many customers exist

### Seeding Performance
- Seeds 50 products on first run (~100-200ms)
- Runs on every startup (idempotent check via `Products.AnyAsync()`)
- For large datasets, consider separate seeding script

### Concurrent Users
- **Cart operations:** Database-locked (serializable isolation)
- **Product browsing:** Read-only, safe to scale
- **Checkout:** Race condition exists (see Known Issues #4)

---

## Deployment Guidance

### Azure App Service
```bash
# Publish
dotnet publish -c Release -o ./publish

# Deploy (requires Azure CLI)
az webapp up --name your-app-name --resource-group your-rg
```

**Required:**
- Change connection string to Azure SQL Database
- Add Application Insights for monitoring
- Replace MockPaymentGateway with real implementation

### Docker (not currently configured)
Sample Dockerfile:
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["RetailMonolith.csproj", "./"]
RUN dotnet restore
COPY . .
RUN dotnet build -c Release -o /app/build

FROM build AS publish
RUN dotnet publish -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "RetailMonolith.dll"]
```

---

## Monitoring & Logging

### Current Logging
- **Console output:** All logs to stdout
- **Log level:** Information for app, Warning for ASP.NET Core

### View Logs
```bash
# Run with verbose logging
dotnet run --verbosity detailed

# Filter to specific namespace
# (Add to appsettings.json)
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "RetailMonolith": "Debug"
    }
  }
}
```

### Production Monitoring
- **Recommended:** Add Application Insights
- **EF Core:** Enable sensitive data logging only in dev
- **Health checks:** Expose after fixing Known Issue #5

---

## Support

### Documentation
- High-Level Design: `/docs/HLD.md`
- Low-Level Design: `/docs/LLD.md`
- Architecture Decisions: `/docs/ADR/`

### Common Questions

**Q: How do I add a new product?**  
A: Manually insert via SQL or extend seeding logic in `AppDbContext.SeedAsync()`

**Q: Can I use PostgreSQL instead of SQL Server?**  
A: Yes, change EF Core provider to `Npgsql.EntityFrameworkCore.PostgreSQL` and update connection string

**Q: Why is checkout only available via API?**  
A: Checkout Razor Page is a placeholder (see Known Issue #7)

**Q: How do I test payment failures?**  
A: Not possible with MockPaymentGateway (see Known Issue #2)
