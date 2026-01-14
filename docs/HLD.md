# High-Level Design (HLD) - Retail Monolith

## System Overview

The Retail Monolith is an ASP.NET Core 8 web application that implements a basic e-commerce system using Razor Pages. The application is a single monolithic deployment that handles all aspects of the retail workflow including product catalog management, shopping cart, order processing, and checkout.

## Architecture Pattern

**Monolithic Architecture**: All functionality is contained within a single deployable unit (web application).

## Components and Modules

The application is organized into the following functional domains:

### 1. Products Domain
- **Purpose**: Manages product catalog and inventory
- **Key Entities**: `Product`, `InventoryItem`
- **Responsibilities**:
  - Display active products to customers
  - Track product details (SKU, name, description, price, category)
  - Maintain inventory quantities per SKU

### 2. Cart Domain
- **Purpose**: Manages shopping cart operations
- **Key Entities**: `Cart`, `CartLine`
- **Service**: `CartService` (implements `ICartService`)
- **Responsibilities**:
  - Create and retrieve customer carts
  - Add items to cart
  - Update cart line quantities
  - Clear cart after checkout

### 3. Orders Domain
- **Purpose**: Manages order lifecycle and history
- **Key Entities**: `Order`, `OrderLine`
- **Responsibilities**:
  - Create orders from cart contents
  - Track order status (Created, Paid, Failed, Shipped)
  - Store order history with line items

### 4. Checkout Domain
- **Purpose**: Orchestrates the checkout process
- **Service**: `CheckoutService` (implements `ICheckoutService`)
- **External Integration**: `IPaymentGateway` (implemented by `MockPaymentGateway`)
- **Responsibilities**:
  - Retrieve customer cart
  - Reserve/decrement inventory
  - Process payment via payment gateway
  - Create order record
  - Clear cart on successful checkout

## Data Stores

### Primary Database
- **Technology**: Microsoft SQL Server (LocalDB for development)
- **ORM**: Entity Framework Core 9.0
- **Connection**: Configured via `appsettings.json` or defaults to `(localdb)\MSSQLLocalDB`
- **Database Name**: `RetailMonolith`

### Data Model
- **DbContext**: `AppDbContext`
- **Tables**:
  - `Products` - Product catalog (unique SKU index)
  - `Inventory` - Stock quantities per SKU (unique SKU index)
  - `Carts` / `CartLines` - Shopping carts with line items
  - `Orders` / `OrderLines` - Order history with line items

### Data Seeding
- Automatic seeding on application startup via `AppDbContext.SeedAsync()`
- Seeds 50 sample products across 6 categories (Apparel, Footwear, Accessories, Electronics, Home, Beauty)
- Random inventory quantities (10-200 units per product)
- Prices in GBP (£5-£105 range)

## External Dependencies

### Payment Gateway
- **Interface**: `IPaymentGateway`
- **Current Implementation**: `MockPaymentGateway`
- **Behavior**: Always returns successful payment with mock transaction ID
- **Production Consideration**: Requires replacement with real payment provider (Stripe, PayPal, etc.)

## API Endpoints

### Razor Pages (UI)
- `/` - Home page
- `/Products` - Product listing (GET: display products, POST: add to cart)
- `/Cart` - Shopping cart view
- `/Checkout` - Checkout page
- `/Orders` - Order history page

### Minimal APIs (Programmatic)
- `POST /api/checkout` - Create order from cart
  - Accepts: No body (uses hardcoded "guest" customer)
  - Returns: `{ id, status, total }`
- `GET /api/orders/{id}` - Retrieve order by ID
  - Returns: Full order with line items or 404

### Health Check
- Service registered: `builder.Services.AddHealthChecks()`
- **Note**: Endpoint not currently mapped in middleware pipeline (tech debt)

## Runtime Environment

### Application Model
- **Framework**: ASP.NET Core 8.0
- **Hosting**: Kestrel web server
- **Configuration**: appsettings.json + environment variables

### Dependency Injection
Services registered at startup:
- `AppDbContext` (Scoped) - EF Core database context
- `ICartService` → `CartService` (Scoped)
- `ICheckoutService` → `CheckoutService` (Scoped)
- `IPaymentGateway` → `MockPaymentGateway` (Scoped)
- Health checks (not exposed)

### Startup Behavior
On application start (in `Program.cs`):
1. Apply pending EF migrations (`Database.MigrateAsync()`)
2. Seed database if empty (`AppDbContext.SeedAsync()`)
3. Configure middleware pipeline
4. Start Kestrel on ports 5000 (HTTP) and 5001 (HTTPS)

### Environment Modes
- **Development**: Exception pages, detailed errors
- **Production**: Error handler at `/Error`, HSTS enabled

## Deployment Model

- Single ASP.NET Core web application
- Requires .NET 8 SDK for build
- Requires SQL Server or LocalDB for database
- No external service dependencies (except mock payment)
- Ready for containerization or Azure App Service deployment

## Security Considerations

- No authentication/authorization implemented
- All users treated as "guest" customer
- HTTPS redirection enabled in production
- Connection strings in appsettings (should use secrets management in production)

## Scalability Constraints

- Single database connection (no read replicas)
- No caching layer
- Synchronous request processing
- Cart and checkout operations lock database rows (potential bottleneck)
- No distributed session state (limits horizontal scaling)
