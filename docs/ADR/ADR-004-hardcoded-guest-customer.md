# ADR-004: Hardcoded "guest" Customer Model

## Status
Accepted (Technical Debt)

## Context
The application handles shopping carts and orders, which require associating data with specific users. A proper multi-tenant system would implement authentication and user management. However, as a demonstration monolith, we needed a simpler approach to focus on the decomposition patterns rather than authentication infrastructure.

## Decision
We will use a **hardcoded "guest" customer ID** for all carts and orders, with no authentication or user management.

## Rationale

### Chosen: Single Hardcoded User

**Implementation Pattern**
```csharp
// Models/Cart.cs
public string CustomerId { get; set; } = "guest";

// Models/Order.cs
public string CustomerId { get; set; } = "guest";

// Throughout codebase
await _cartService.GetCartWithLinesAsync("guest");
await svc.CheckoutAsync("guest", "tok_test");
```

**Benefits**
- **Minimal complexity**: No authentication middleware, login pages, or session management
- **Focus on domain logic**: Demonstrates cart/order flows without auth distraction
- **Fast development**: Can test checkout immediately without user registration
- **Clear technical debt**: Obvious that this is not production-ready

### Alternatives Considered

**Cookie-Based Session ID**
- Use `HttpContext.Session.Id` as customer ID
- More realistic multi-user behavior
- Requires session state management (in-memory or distributed)
- Still no authentication, users cannot return to their cart

**Anonymous User with GUID**
- Generate GUID for each browser session
- Store in cookie or localStorage
- Better simulation of multiple users
- Requires cookie/session infrastructure

**Full Authentication (ASP.NET Identity)**
- Proper user accounts with login
- Requires database tables (AspNetUsers, AspNetRoles, etc.)
- Login/register pages needed
- Significant scope increase for demonstration app

**No Customer ID**
- Store cart in browser only (localStorage)
- Cannot persist cart server-side
- Checkout would require sending entire cart in request
- Not realistic for order tracking

## Consequences

### Positive
- ✅ **Simplicity**: Zero authentication code, configuration, or UI
- ✅ **Testability**: Known, consistent customer ID for all tests
- ✅ **Focus**: Keeps attention on monolith structure, not auth patterns
- ✅ **Obvious limitation**: Cannot be mistaken for production-ready code

### Negative
- ❌ **Single-tenant behavior**: All users share the same cart and order history
- ❌ **Data collision**: Concurrent users will interfere with each other's carts
- ❌ **No isolation**: Cannot test multi-user scenarios
- ❌ **No personalization**: Cannot track user preferences or history
- ❌ **Security gap**: No authorization checks (anyone can view any order)

## Impact on Application Behavior

### Cart Operations
- All users modify the **same cart**
- If User A adds item and User B adds item, both see both items
- Clearing cart affects all users

**Evidence in Code**
```csharp
// Pages/Products/Index.cshtml.cs line 49
await _cartService.AddToCartAsync("guest", productId);

// Pages/Cart/Index.cshtml.cs line 32
var cart = await _cartService.GetCartWithLinesAsync("guest");
```

### Checkout Operations
- All orders belong to "guest"
- Cannot query "my orders" vs "all orders" (they're the same)

**Evidence in Code**
```csharp
// Program.cs line 53 (API endpoint)
var order = await svc.CheckoutAsync("guest", "tok_test");
```

### Order Queries
```csharp
// Program.cs lines 57-63
app.MapGet("/api/orders/{id:int}", async (int id, AppDbContext db) =>
{
    var order = await db.Orders.Include(o => o.Lines)
        .SingleOrDefaultAsync(o => o.Id == id);
    return order is null ? Results.NotFound() : Results.Ok(order);
});
```

**Issue**: No customer ID validation. Any user can retrieve any order by ID.

## Technical Debt Items

### 1. Missing Customer Table
No `Customer` entity exists. The `CustomerId` is a string field with no referential integrity.

**Implication**: Cannot enforce foreign key constraints or query customer data.

### 2. No Session Management
Each HTTP request is independent. No session cookie or context tracking.

**Implication**: Cannot maintain cart state across browser sessions (though DB persistence helps).

### 3. No Authorization
No `[Authorize]` attributes or policy-based checks.

**Implication**: All endpoints are publicly accessible without authentication.

### 4. Hardcoded in Multiple Layers
- Models (default values)
- Page Models (all methods)
- Services (none, they accept `customerId` parameter ✅)
- API endpoints

**Good**: Services use `customerId` parameter, allowing future injection of real ID.

### 5. Comment Evidence of Known Issue
```csharp
// Pages/Cart/Index.cshtml.cs lines 22-24
//map the cart state in memory without mapping directly to the database
// Each line represents an item in the cart with its name, quantity, and price
//is this a good way to represent a cart in memory?
```

This comment suggests awareness that the cart design is questionable, possibly related to the "guest" limitation.

## Migration Path to Real Authentication

When transitioning to production, follow these steps:

### 1. Add ASP.NET Core Identity
```bash
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
```

Update `AppDbContext`:
```csharp
public class AppDbContext : IdentityDbContext<ApplicationUser>
{
    // ... existing DbSets
}
```

### 2. Extract Customer ID from Claims
```csharp
// Extension method
public static string GetCustomerId(this ClaimsPrincipal user)
{
    return user.FindFirstValue(ClaimTypes.NameIdentifier) 
           ?? throw new UnauthorizedAccessException();
}
```

### 3. Update Page Models
```csharp
// Before
await _cartService.AddToCartAsync("guest", productId);

// After
var customerId = User.GetCustomerId();
await _cartService.AddToCartAsync(customerId, productId);
```

### 4. Protect API Endpoints
```csharp
app.MapPost("/api/checkout", [Authorize] async (ClaimsPrincipal user, ICheckoutService svc) =>
{
    var customerId = user.GetCustomerId();
    var order = await svc.CheckoutAsync(customerId, "tok_test");
    return Results.Ok(new { order.Id, order.Status, order.Total });
});
```

### 5. Enforce Order Access Control
```csharp
app.MapGet("/api/orders/{id:int}", [Authorize] async (
    int id, 
    ClaimsPrincipal user, 
    AppDbContext db) =>
{
    var customerId = user.GetCustomerId();
    var order = await db.Orders
        .Include(o => o.Lines)
        .SingleOrDefaultAsync(o => o.Id == id && o.CustomerId == customerId);
    
    return order is null ? Results.NotFound() : Results.Ok(order);
});
```

### 6. Data Migration
Convert existing "guest" data:
```sql
-- Option 1: Delete test data
DELETE FROM Orders WHERE CustomerId = 'guest';
DELETE FROM Carts WHERE CustomerId = 'guest';

-- Option 2: Assign to a test user
UPDATE Orders SET CustomerId = 'test-user-id' WHERE CustomerId = 'guest';
UPDATE Carts SET CustomerId = 'test-user-id' WHERE CustomerId = 'guest';
```

## Testing Impact

### Current State
- Tests can use "guest" as a known constant
- No need to mock `IHttpContextAccessor` or claims
- Simpler test setup

### With Real Auth
- Tests need to mock `ClaimsPrincipal`
- Integration tests require authenticated test users
- More complex but more realistic

## Documentation Accuracy

### README.md Claims
The README states:
> "50 sample seeded products with inventory"

But doesn't mention that all users share the same cart. This is acceptable for a demo but should be clarified.

### Suggested Addition to README
```markdown
## Limitations

⚠️ **Single User Mode**: This demo uses a hardcoded "guest" user. All visitors share the same shopping cart and order history. This is intentional for demonstration purposes and must be replaced with proper authentication before production use.
```

## Notes
The "guest" customer approach is acceptable **only** for this demonstration/prototype context. It allows the application to showcase the domain logic (products, cart, checkout, orders) without the distraction of authentication infrastructure.

However, this is **critical technical debt** that must be addressed before any production deployment. The good news is that the service layer (`ICartService`, `ICheckoutService`) already accepts `customerId` as a parameter, making it easier to inject real user IDs when authentication is implemented.

## Related ADRs
- ADR-001: Razor Pages (no login UI implemented)
- ADR-003: Mock Payment Gateway (no PCI compliance needed without real users)
