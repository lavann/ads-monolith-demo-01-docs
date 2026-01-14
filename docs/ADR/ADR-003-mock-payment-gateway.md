# ADR-003: Use Mock Payment Gateway for Development

## Status
Accepted (Interim)

## Context
The checkout process requires integration with a payment provider to charge customers. Real payment gateways (Stripe, PayPal, Braintree) require API credentials, webhook configuration, and external network access. We needed a solution for local development and testing that doesn't require these dependencies.

## Decision
We will use a **Mock Payment Gateway** implementation that always returns successful payments for development and demonstration purposes.

## Rationale

### Chosen: Mock Implementation

**Implementation Details**
```csharp
// Services/MockPaymentGateway.cs
public class MockPaymentGateway : IPaymentGateway
{
    public Task<PaymentResult> ChargeAsync(PaymentRequest req, CancellationToken ct = default)
    {
        return Task.FromResult(new PaymentResult(
            Succeeded: true, 
            ProviderRef: $"MOCK-{Guid.NewGuid():N}", 
            Error: null
        ));
    }
}
```

**Benefits**
- **No external dependencies**: Works offline, no API keys needed
- **Predictable behavior**: Always succeeds, simplifies testing checkout flow
- **Fast execution**: Synchronous operation wrapped in `Task.FromResult`, no network latency
- **Simple debugging**: No webhook handling or async callbacks

**Interface Design**
- `IPaymentGateway` provides clean abstraction boundary
- Request/Result pattern using record types
- Supports cancellation token for future timeout scenarios

### Alternatives Considered

**Stripe Test Mode**
- Requires API key and network access
- Provides realistic test card numbers and scenarios
- Better for integration testing but adds complexity
- Requires webhook endpoint for payment confirmation

**Fake HTTP Responses (WireMock/MockServer)**
- More realistic HTTP client behavior
- Complex setup for simple use case
- Requires managing separate mock server process

**In-Memory Test Double in Tests Only**
- Would require real implementation for dev environment
- Forces developers to configure payment provider credentials

## Consequences

### Positive
- ✅ **Zero configuration**: App runs immediately after clone
- ✅ **Deterministic testing**: Checkout always succeeds, no flaky tests
- ✅ **Clear separation**: `IPaymentGateway` interface enforces contract
- ✅ **Easy swap**: Can replace with real provider without changing `CheckoutService`

### Negative
- ❌ **No failure testing**: Cannot test declined cards, timeouts, or error scenarios
- ❌ **Production risk**: Mock must be replaced before deployment (easy to forget)
- ❌ **Incomplete contract**: Real gateways return more metadata (card brand, last 4 digits, AVS results)
- ❌ **No webhook simulation**: Real gateways use async webhooks for payment confirmation

## Current Integration Points

### Service Registration
```csharp
// Program.cs line 16
builder.Services.AddScoped<IPaymentGateway, MockPaymentGateway>();
```

**Risk**: No environment-based switching logic. Mock is registered for all environments.

**Recommendation**: Use configuration to select implementation:
```csharp
if (builder.Environment.IsDevelopment())
{
    builder.Services.AddScoped<IPaymentGateway, MockPaymentGateway>();
}
else
{
    builder.Services.AddScoped<IPaymentGateway, StripePaymentGateway>();
}
```

### Usage in CheckoutService
```csharp
// Services/CheckoutService .cs line 35
var pay = await _payments.ChargeAsync(new(total, "GBP", paymentToken), ct);
var status = pay.Succeeded ? "Paid" : "Failed";
```

**Note**: Payment token (`paymentToken`) is not used by mock implementation. Real gateway would validate and charge the token.

### Hardcoded Token in API
```csharp
// Program.cs line 53
var order = await svc.CheckoutAsync("guest", "tok_test");
```

**Issue**: API endpoint hardcodes `"tok_test"` token. Real implementation would accept token from request body.

## Future Production Requirements

Before deploying to production, the following changes are required:

### 1. Implement Real Payment Gateway
Example for Stripe:
```csharp
public class StripePaymentGateway : IPaymentGateway
{
    private readonly StripeClient _stripe;
    
    public async Task<PaymentResult> ChargeAsync(PaymentRequest req, CancellationToken ct)
    {
        var options = new PaymentIntentCreateOptions
        {
            Amount = (long)(req.Amount * 100), // Convert to cents
            Currency = req.Currency.ToLower(),
            PaymentMethod = req.Token,
            Confirm = true
        };
        
        try
        {
            var intent = await _stripe.PaymentIntents.CreateAsync(options, cancellationToken: ct);
            return new PaymentResult(
                Succeeded: intent.Status == "succeeded",
                ProviderRef: intent.Id,
                Error: intent.Status != "succeeded" ? intent.CancellationReason : null
            );
        }
        catch (StripeException ex)
        {
            return new PaymentResult(false, null, ex.Message);
        }
    }
}
```

### 2. Add Configuration
`appsettings.json`:
```json
{
  "Stripe": {
    "ApiKey": "sk_test_...",
    "WebhookSecret": "whsec_..."
  }
}
```

### 3. Accept Payment Token from Client
Modify API to accept payment details:
```csharp
app.MapPost("/api/checkout", async (CheckoutRequest req, ICheckoutService svc) =>
{
    var order = await svc.CheckoutAsync(req.CustomerId, req.PaymentToken);
    return Results.Ok(new { order.Id, order.Status, order.Total });
});

record CheckoutRequest(string CustomerId, string PaymentToken);
```

### 4. Handle Payment Failure
Update `CheckoutService` to rollback inventory on payment failure:
```csharp
if (!pay.Succeeded)
{
    // Restore inventory
    foreach (var line in cart.Lines)
    {
        var inv = await _db.Inventory.SingleAsync(i => i.Sku == line.Sku, ct);
        inv.Quantity += line.Quantity; // Rollback
    }
}
```

### 5. Implement Webhooks
Add endpoint to handle async payment confirmations:
```csharp
app.MapPost("/webhooks/stripe", async (HttpRequest req, IStripeWebhookHandler handler) =>
{
    var signature = req.Headers["Stripe-Signature"];
    var json = await new StreamReader(req.Body).ReadToEndAsync();
    await handler.ProcessWebhookAsync(json, signature);
    return Results.Ok();
});
```

## Testing Strategy

### Current State
- Mock allows happy-path testing only
- No test coverage for payment failures

### Recommended Approach
1. **Unit tests**: Use mock implementation (current approach is fine)
2. **Integration tests**: Use Stripe test mode with test cards:
   - `4242 4242 4242 4242` - Success
   - `4000 0000 0000 0002` - Declined
   - `4000 0000 0000 9995` - Insufficient funds
3. **E2E tests**: Use Stripe test mode with webhook simulation

## Security Considerations

### Current Issues
- ✅ Payment token not logged or stored (good)
- ⚠️ No PCI compliance needed (mock doesn't handle real card data)
- ⚠️ Token hardcoded in API endpoint (not sent by client)

### Production Requirements
- Store only tokenized payment methods, never raw card numbers
- Use HTTPS for all payment API calls
- Validate webhook signatures to prevent forgery
- Log payment attempts (success/failure) for audit trail
- Implement rate limiting on checkout endpoint

## Notes
The mock payment gateway is appropriate for the "demonstration monolith" purpose. It allows the checkout flow to be tested without external dependencies. However, **it must not be deployed to production** as it provides no actual payment processing.

Consider using feature flags or environment-based configuration to ensure the mock is never accidentally used in production.
