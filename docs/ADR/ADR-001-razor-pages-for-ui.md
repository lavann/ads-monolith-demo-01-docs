# ADR-001: Use Razor Pages for Web UI

## Status
Accepted

## Context
The Retail Monolith requires a web UI for customers to browse products, manage their cart, and complete checkout. We needed to choose a UI framework that integrates well with ASP.NET Core and allows rapid development.

## Decision
We will use **ASP.NET Core Razor Pages** as the UI framework for the application.

## Rationale

### Chosen: Razor Pages
- **Server-side rendering**: Simple deployment model, no separate frontend build process
- **Page-centric structure**: Each page (Products, Cart, Checkout) maps to a route and file, making it easy to locate code
- **Tight integration**: Native ASP.NET Core integration with minimal configuration
- **Rapid development**: Code-behind model allows business logic close to the view
- **Form handling**: Built-in model binding and validation for POST operations

### Alternatives Considered

**MVC Pattern**
- More ceremony required (Controllers, Views, Models separated)
- Better for complex applications with many shared views
- Overkill for page-focused e-commerce flow

**Blazor Server**
- Requires WebSocket for SignalR connection
- More complex state management
- Better for highly interactive dashboards, not simple CRUD

**SPA (React/Angular) + Web API**
- Requires separate frontend build toolchain
- More complexity in deployment and development environment
- Better for mobile app integration or heavy client-side logic
- Not needed for current requirements

## Consequences

### Positive
- Faster initial development with simpler architecture
- No JavaScript build step or npm dependencies (except vendored libs)
- Easy for .NET developers to understand and maintain
- Built-in CSRF protection for forms

### Negative
- **Limited interactivity**: Full page reloads on navigation (could add AJAX later)
- **Server-side sessions**: Cart state stored in database, not client-side (adds DB load)
- **Coupling**: Page models can access services directly, risk bypassing business logic
- **Not API-first**: Requires parallel API endpoints for programmatic access (see `/api/checkout`)

### Observed Issues in Codebase
- Some Page Models bypass service layer and access `AppDbContext` directly (e.g., `ProductsIndexModel` lines 32-40)
- Empty page models (`Checkout/IndexModel`, `Orders/IndexModel`) suggest incomplete UI implementation
- Checkout flow implemented as API only, no Razor Page implementation

## Notes
The decision to use Razor Pages is appropriate for the current "monolith demonstration" purpose. For a production e-commerce system, consider:
- Adding client-side validation and progressive enhancement
- Implementing AJAX cart updates to avoid full page reloads
- Adding proper session management for cart state
- Completing the checkout UI flow in Razor Pages (currently API-only)
