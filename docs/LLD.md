# Low-Level Design (LLD) - Retail Monolith

## Overview

This document details the key classes, services, and request flows within the Retail Monolith application, highlighting areas of coupling and technical considerations.

## Domain Organization

### Products Module

#### Key Classes

**`Product`** (`Models/Product.cs`)
- Properties: `Id`, `Sku`, `Name`, `Description`, `Price`, `Currency`, `IsActive`, `Category`
- Unique constraint on `Sku`
- Simple POCO with no behavior

**`InventoryItem`** (`Models/InventoryItem.cs`)
- Properties: `Id`, `Sku`, `Quantity`
- Unique constraint on `Sku`
- Tracks available stock per product SKU

#### Page Model

**`Pages/Products/IndexModel`** (`Pages/Products/Index.cshtml.cs`)
- Dependencies: `AppDbContext`, `ICartService`
- `OnGetAsync()`: Queries active products from database
- `OnPostAsync(int productId)`: 
  - Adds product to cart via `CartService`
  - Contains duplicate cart creation logic (tech debt - see Hotspots)
  - Redirects to `/Cart`

---

### Cart Module

#### Key Classes

**`Cart`** (`Models/Cart.cs`)
- Properties: `Id`, `CustomerId`, `Lines` (collection of `CartLine`)
- Default `CustomerId` = "guest"
- Represents a shopping cart session

**`CartLine`** (`Models/Cart.cs`)
- Properties: `Id`, `CartId`, `Cart`, `Sku`, `Name`, `UnitPrice`, `Quantity`
- Child entity of `Cart`
- Stores snapshot of product details at time of add

#### Service Layer

**`ICartService`** (`Services/ICartService.cs`)
Interface defining cart operations:
- `GetOrCreateCartAsync(customerId)` - Retrieve or create cart
- `AddToCartAsync(customerId, productId, quantity)` - Add item to cart
- `GetCartWithLinesAsync(customerId)` - Get cart with line items loaded
- `ClearCartAsync(customerId)` - Delete cart and lines

**`CartService`** (`Services/CartService.cs`)
- Dependencies: `AppDbContext`
- Handles cart CRUD operations
- Checks for duplicate products in cart (updates quantity if exists)
- Snapshots product details (name, price) into cart lines

#### Page Model

**`Pages/Cart/IndexModel`** (`Pages/Cart/Index.cshtml.cs`)
- Dependencies: `ICartService`
- `Lines` property: In-memory tuple list `(Name, Quantity, Price)`
- `Total` property: Calculated sum of line totals
- `OnGetAsync()`: Loads cart and projects to tuples (tech debt - see Hotspots)

---

### Orders Module

#### Key Classes

**`Order`** (`Models/Order.cs`)
- Properties: `Id`, `CreatedUtc`, `CustomerId`, `Status`, `Total`, `Lines`
- Status values: "Created", "Paid", "Failed", "Shipped"
- Immutable after creation (no update logic)

**`OrderLine`** (`Models/Order.cs`)
- Properties: `Id`, `OrderId`, `Order`, `Sku`, `Name`, `UnitPrice`, `Quantity`
- Child entity of `Order`
- Copied from `CartLine` during checkout

#### Page Model

**`Pages/Orders/IndexModel`** (`Pages/Orders/Index.cshtml.cs`)
- Currently empty implementation (placeholder)
- No order listing or detail views implemented

---

### Checkout Module

#### Service Layer

**`ICheckoutService`** (`Services/ICheckoutService.cs`)
Interface defining checkout operation:
- `CheckoutAsync(customerId, paymentToken)` - Process cart checkout

**`CheckoutService`** (`Services/CheckoutService .cs`)
- Dependencies: `AppDbContext`, `IPaymentGateway`
- Orchestrates multi-step checkout process (see Request Flow below)
- Single database transaction (no explicit transaction management - relies on SaveChanges)

**`IPaymentGateway`** (`Services/IPaymentGateway.cs`)
- Records: `PaymentRequest`, `PaymentResult`
- Abstraction for payment processing
- Current implementation: `MockPaymentGateway` (always succeeds)

#### Page Model

**`Pages/Checkout/IndexModel`** (`Pages/Checkout/Index.cshtml.cs`)
- Currently empty implementation (placeholder)
- Checkout initiated via API endpoint, not Razor Page

---

## Key Request Flows

### 1. Browse Products Flow

```
User → GET /Products
  ↓
ProductsIndexModel.OnGetAsync()
  ↓
AppDbContext.Products.Where(p => p.IsActive).ToListAsync()
  ↓
Render product list
```

**Database Queries**: 1 (SELECT Products WHERE IsActive = 1)

---

### 2. Add to Cart Flow

```
User → POST /Products (productId)
  ↓
ProductsIndexModel.OnPostAsync(productId)
  ↓
[Duplicate Logic Issue]
  ├─ Direct DB query for cart (unused code)
  └─ CartService.AddToCartAsync("guest", productId)
       ↓
     GetOrCreateCartAsync("guest")
       ↓
     Load product from DB
       ↓
     Check for existing CartLine with same SKU
       ↓
     Update quantity OR add new CartLine
       ↓
     SaveChangesAsync()
  ↓
Redirect to /Cart
```

**Database Queries**: 3-4
- SELECT Cart + CartLines (GetOrCreateCart)
- SELECT Product (FindAsync)
- INSERT/UPDATE CartLine
- Optionally INSERT Cart if new

**Hotspot**: Duplicate cart lookup logic in page model (see Coupling section)

---

### 3. View Cart Flow

```
User → GET /Cart
  ↓
CartIndexModel.OnGetAsync()
  ↓
CartService.GetCartWithLinesAsync("guest")
  ↓
Query: Carts.Include(c => c.Lines).FirstOrDefault(...)
  ↓
Project to tuple list: (Name, Quantity, Price)
  ↓
Calculate Total (in-memory)
  ↓
Render cart view
```

**Database Queries**: 1 (SELECT Cart + CartLines)

**Tech Debt**: Unnecessary projection to tuples instead of using CartLine entities directly

---

### 4. Checkout Flow (API)

```
Client → POST /api/checkout
  ↓
CheckoutService.CheckoutAsync("guest", "tok_test")
  ↓
  [Step 1] Load cart with lines
    ↓ Query: Carts.Include(c => c.Lines).SingleOrDefault(...)
    ↓ Throw if cart not found
  ↓
  [Step 2] Calculate total
  ↓
  [Step 3] Reserve inventory (optimistic locking)
    ↓ For each CartLine:
        Query: Inventory.Single(i => i.Sku == line.Sku)
        Check: inv.Quantity >= line.Quantity
        Decrement: inv.Quantity -= line.Quantity
        Throw if out of stock
  ↓
  [Step 4] Charge payment
    ↓ PaymentGateway.ChargeAsync(PaymentRequest)
    ↓ Set order status: "Paid" or "Failed"
  ↓
  [Step 5] Create order
    ↓ Map CartLines → OrderLines
    ↓ Add Order to DbContext
  ↓
  [Step 6] Clear cart
    ↓ Remove CartLines from DbContext
  ↓
  SaveChangesAsync() - commits all changes
  ↓
Return order summary: { id, status, total }
```

**Database Queries**: 
- 1 (SELECT Cart + CartLines)
- N (SELECT Inventory per cart line)
- 1 (INSERT Order + OrderLines, DELETE CartLines)

**Critical Path**: Inventory decrements occur before payment, creating risk of stock reservation without payment

---

### 5. Get Order Flow (API)

```
Client → GET /api/orders/{id}
  ↓
Minimal API handler (Program.cs)
  ↓
Query: Orders.Include(o => o.Lines).SingleOrDefault(o => o.Id == id)
  ↓
Return order or 404
```

**Database Queries**: 1 (SELECT Order + OrderLines by ID)

---

## Areas of Coupling

### 1. Direct Database Access in Page Models

**Issue**: `ProductsIndexModel.OnPostAsync()` directly accesses `AppDbContext` to query cart
- **Location**: `Pages/Products/Index.cshtml.cs:32-40`
- **Problem**: Duplicates cart creation logic that exists in `CartService`
- **Impact**: Inconsistent cart handling, harder to maintain
- **Recommendation**: Remove direct DB access, rely solely on `CartService`

### 2. Hardcoded "guest" Customer ID

**Issue**: Customer ID "guest" is hardcoded throughout the application
- **Locations**: 
  - `Cart.CustomerId` default value
  - `Order.CustomerId` default value
  - All Page Models and API endpoints
- **Problem**: No multi-user support, cannot distinguish customers
- **Impact**: Single-tenant behavior, cannot scale to real users
- **Recommendation**: Implement authentication and pass authenticated user ID

### 3. Tight Coupling Between Checkout and Inventory

**Issue**: `CheckoutService` directly manipulates inventory quantities
- **Location**: `Services/CheckoutService .cs:27-32`
- **Problem**: No inventory domain service, business logic embedded in checkout
- **Impact**: Cannot reuse inventory logic, difficult to add reservation/backorder features
- **Recommendation**: Extract `IInventoryService` with reserve/commit/rollback operations

### 4. Payment Gateway Abstraction Leak

**Issue**: Payment token "tok_test" hardcoded in API endpoint
- **Location**: `Program.cs:53`
- **Problem**: API contract doesn't accept payment details from client
- **Impact**: Cannot process real payments without changing API contract
- **Recommendation**: Accept payment token in request body

### 5. Missing Transaction Management

**Issue**: Checkout process has no explicit transaction boundaries
- **Location**: `CheckoutService.CheckoutAsync()`
- **Problem**: Multiple database operations rely on implicit EF transaction
- **Impact**: If payment succeeds but SaveChanges fails, inventory is decremented but order not created
- **Recommendation**: Wrap checkout in explicit `DbContext.Database.BeginTransactionAsync()`

---

## Technical Hotspots

### 1. Duplicate Cart Logic in ProductsIndexModel

**Severity**: Medium  
**Location**: `Pages/Products/Index.cshtml.cs:32-48`

Lines 32-40 create cart directly via DbContext, then line 49 calls `CartService.AddToCartAsync()` which does the same thing. The direct DbContext code is effectively dead code but creates maintenance confusion.

```csharp
// Lines 32-40: Unused cart creation
var cart = await _db.Carts
    .Include(c => c.Lines)
    .SingleOrDefaultAsync(c => c.CustomerId == "guest")
    ?? new Models.Cart { CustomerId = "guest" };

if(cart.Id == 0)
{
    _db.Carts.Add(cart);
};

// Line 49: Actual cart operation (makes above code redundant)
await _cartService.AddToCartAsync("guest", productId);
```

**Recommendation**: Remove lines 27-48, keep only `CartService` call.

---

### 2. Unnecessary Tuple Projection in CartIndexModel

**Severity**: Low  
**Location**: `Pages/Cart/Index.cshtml.cs:25-35`

Cart view projects `CartLine` entities to tuples `(Name, Quantity, Price)`, adding unnecessary mapping layer.

```csharp
public List<(string Name, int Quantity, decimal Price)> Lines { get; set; } = new();

public async Task OnGetAsync()
{
    var cart = await _cartService.GetCartWithLinesAsync("guest");
    Lines = cart.Lines
        .Select(line => (line.Name, line.Quantity, line.UnitPrice))
        .ToList();
}
```

**Impact**: Comment on line 22-24 questions this design choice.

**Recommendation**: Use `List<CartLine>` directly or create a proper DTO class if needed.

---

### 3. Inventory Race Condition

**Severity**: High  
**Location**: `Services/CheckoutService .cs:27-32`

Inventory check and decrement uses optimistic concurrency without row locking.

```csharp
var inv = await _db.Inventory.SingleAsync(i => i.Sku == line.Sku, ct);
if (inv.Quantity < line.Quantity) throw new InvalidOperationException($"Out of stock: {line.Sku}");
inv.Quantity -= line.Quantity;
```

**Problem**: Two concurrent checkouts can both read the same quantity, both pass the check, and both decrement, leading to negative inventory.

**Recommendation**: 
- Use row-level locking (`FromSqlRaw` with `WITH (UPDLOCK)`)
- Add concurrency token to `InventoryItem` entity
- Implement saga pattern for distributed transactions

---

### 4. Payment Before Inventory Reservation

**Severity**: Medium  
**Location**: `Services/CheckoutService .cs:27-35`

Flow decrements inventory, then charges payment. If payment fails, inventory is already decremented.

**Problem**: Failed payments leave inventory in decremented state (though order status is "Failed")

**Recommendation**: 
- Option 1: Charge payment first, then decrement inventory on success
- Option 2: Implement compensating transaction to restore inventory on payment failure
- Option 3: Add explicit "reserved" state to inventory

---

### 5. Missing Health Check Endpoint

**Severity**: Low  
**Location**: `Program.cs:19`

Health checks service is registered but endpoint not mapped.

```csharp
builder.Services.AddHealthChecks(); // Line 19
// Missing: app.MapHealthChecks("/health");
```

**Impact**: README.md claims `/health` endpoint exists but returns 404.

**Recommendation**: Add `app.MapHealthChecks("/health");` after line 47.

---

### 6. Empty Page Models

**Severity**: Low  
**Locations**: 
- `Pages/Checkout/Index.cshtml.cs`
- `Pages/Orders/Index.cshtml.cs`

Both page models have empty `OnGet()` methods, suggesting incomplete UI implementation.

**Impact**: Users cannot checkout or view orders via web UI, must use APIs.

**Recommendation**: Implement UI checkout flow or remove placeholder pages.

---

## Service Dependencies Graph

```
┌─────────────────┐
│ Razor Pages     │
│ (UI Layer)      │
└────────┬────────┘
         │
         ├─────────────┐
         │             │
         ▼             ▼
┌─────────────┐  ┌──────────────┐
│ CartService │  │CheckoutService│
└──────┬──────┘  └───────┬───────┘
       │                 │
       │         ┌───────┴────────┐
       │         │                │
       ▼         ▼                ▼
┌────────────────────┐  ┌─────────────────┐
│   AppDbContext     │  │ IPaymentGateway │
│   (EF Core)        │  │ (External)      │
└────────────────────┘  └─────────────────┘
         │
         ▼
┌────────────────────┐
│   SQL Server DB    │
└────────────────────┘
```

**Notes**:
- No cross-domain service dependencies (good separation)
- All services depend on `AppDbContext` (database-centric design)
- `CheckoutService` depends on both `AppDbContext` and `IPaymentGateway`
- Page Models bypass service layer in some cases (Products module)

---

## Technology Stack Details

### Frameworks & Libraries
- **ASP.NET Core 8.0** - Web framework
- **Entity Framework Core 9.0.9** - ORM
- **Microsoft.EntityFrameworkCore.SqlServer 9.0.9** - SQL Server provider
- **Microsoft.AspNetCore.Diagnostics.HealthChecks 2.2.0** - Health monitoring
- **Microsoft.Extensions.Http.Polly 9.0.9** - Resilience (not actively used)

### Language Features Used
- **C# 12** (implicit usings, nullable reference types)
- **Record types** for DTOs (`PaymentRequest`, `PaymentResult`)
- **Top-level statements** in `Program.cs`
- **Async/await** throughout
- **Dependency injection** via constructor injection

---

## Testing Considerations

### Testability Issues
1. **Hardcoded "guest"**: Difficult to test multi-user scenarios
2. **Missing interfaces**: `ProductsIndexModel` uses concrete `AppDbContext`, not abstracted
3. **Side effects in Page Models**: `OnPostAsync` performs business logic (should be in service)
4. **No integration tests**: No test project found in solution

### Recommended Test Strategy
- Unit tests for `CartService`, `CheckoutService`, `MockPaymentGateway`
- Integration tests for database operations (EF Core in-memory provider)
- End-to-end tests for checkout flow with real DB
- Mock `IPaymentGateway` for service tests
