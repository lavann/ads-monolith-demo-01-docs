# Documentation Index

This directory contains comprehensive documentation for the Retail Monolith application as it currently exists.

## Quick Navigation

### 📘 [High-Level Design (HLD)](./HLD.md)
System architecture, components, data stores, and runtime environment. Start here for a big-picture understanding.

**Key Sections:**
- System overview and architecture pattern
- Domain boundaries (Products, Cart, Orders, Checkout)
- Data model and EF Core setup
- API endpoints and external dependencies
- Deployment model and scalability constraints

---

### 📗 [Low-Level Design (LLD)](./LLD.md)
Detailed class structure, request flows, and technical analysis. Read this for implementation details.

**Key Sections:**
- Key classes and services per domain module
- Main request flows (with database query counts)
- Areas of coupling between components
- Technical hotspots and code quality issues
- Service dependency graph

---

### 📙 [Architecture Decision Records (ADRs)](./ADR/)
Documents key architectural choices made in the codebase, with rationale and consequences.

1. **[ADR-001: Razor Pages for UI](./ADR/ADR-001-razor-pages-for-ui.md)**  
   Why we chose Razor Pages over MVC, Blazor, or SPA frameworks

2. **[ADR-002: Entity Framework Core with LocalDB](./ADR/ADR-002-ef-core-with-localdb.md)**  
   Why we use EF Core as ORM and LocalDB for development

3. **[ADR-003: Mock Payment Gateway](./ADR/ADR-003-mock-payment-gateway.md)**  
   Why payments are mocked and what's needed for production

4. **[ADR-004: Hardcoded "guest" Customer](./ADR/ADR-004-hardcoded-guest-customer.md)**  
   Why there's no authentication and how to add it

---

### 📕 [Runbook](./Runbook.md)
Operational guide for building, running, and troubleshooting the application. Essential for developers.

**Key Sections:**
- Prerequisites and initial setup
- Common commands (build, run, database operations)
- Configuration options
- Troubleshooting guide
- Known issues and technical debt items
- Deployment guidance

---

## Documentation Scope

This documentation reflects the **current state** of the codebase as of January 2026, including:
- ✅ Actual code structure and behavior
- ✅ Known technical debt and limitations
- ✅ Identified coupling and hotspots
- ✅ Operational procedures

This documentation does **not** include:
- ❌ Speculative future designs
- ❌ Refactoring recommendations (beyond noting issues)
- ❌ Behavior changes or code modifications

---

## Key Findings Summary

### Domain Boundaries Identified

| Domain | Entities | Services | UI |
|--------|----------|----------|-----|
| **Products** | Product, InventoryItem | None (direct DB access) | `/Products` |
| **Cart** | Cart, CartLine | CartService | `/Cart` |
| **Orders** | Order, OrderLine | None | `/Orders` (placeholder) |
| **Checkout** | N/A | CheckoutService | `/api/checkout` (API only) |

### Critical Technical Debt

1. **No Authentication** - All users are "guest" (production blocker)
2. **Mock Payments** - No real payment processing (production blocker)
3. **Inventory Race Condition** - Concurrent checkouts can oversell (high severity)
4. **No Transaction Management** - Checkout can leave inconsistent state (medium severity)
5. **Missing Health Check Endpoint** - Service registered but not mapped (low severity)

### Areas of Coupling

1. **Products → Cart**: Direct DbContext access bypasses CartService
2. **Checkout → Inventory**: No inventory service abstraction
3. **All Modules → "guest"**: Hardcoded customer ID throughout
4. **All Services → AppDbContext**: No repository pattern

---

## Reading Paths

### For Business Stakeholders
1. Start with [HLD](./HLD.md) - System Overview and Components sections
2. Review "Known Issues" in [Runbook](./Runbook.md)
3. Skim [ADR-004](./ADR/ADR-004-hardcoded-guest-customer.md) to understand single-user limitation

### For Developers (New to Codebase)
1. Read [HLD](./HLD.md) in full
2. Read [Runbook](./Runbook.md) - Prerequisites and Initial Setup
3. Run the application following Runbook instructions
4. Read [LLD](./LLD.md) - Domain Organization and Request Flows
5. Explore code with LLD as reference

### For Architects (Planning Decomposition)
1. Read [HLD](./HLD.md) - Components and Data Stores
2. Read [LLD](./LLD.md) - Areas of Coupling and Technical Hotspots
3. Review all [ADRs](./ADR/) to understand design constraints
4. Study Service Dependencies Graph in LLD
5. Identify boundaries for microservice extraction

### For QA/Testers
1. Read [Runbook](./Runbook.md) - Initial Setup
2. Review [HLD](./HLD.md) - API Endpoints section
3. Check [Runbook](./Runbook.md) - Known Issues for expected behaviors
4. Read [ADR-003](./ADR/ADR-003-mock-payment-gateway.md) to understand payment limitations

---

## Maintenance

This documentation was generated as part of **Issue #01: Document Current Monolith**.

As the codebase evolves, documentation should be updated to reflect:
- New features or modules added
- Changes to domain boundaries
- Resolution of known issues
- New architectural decisions (add new ADRs)
- Changes to build/run procedures

---

## Questions or Issues?

If you find inaccuracies in this documentation or need clarification:
1. Check the actual code (documentation reflects code, not intentions)
2. Refer to commit history for context on specific files
3. Open a GitHub issue with the `documentation` label
