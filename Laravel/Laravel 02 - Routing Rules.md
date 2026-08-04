# Laravel 02 - Routing Rules

Routing in utilitarian architecture follows a set of principles that maintain semantic clarity, domain focus, and separation of concerns.

> **Note:** Examples below pass an Action/Command/Query class directly as the route handler (e.g. `Route::get(..., ShowPaymentAction::class)`). This requires that class to implement `__invoke()` — the base classes do not provide one. Invokable syntax is optional, not a requirement (see Laravel 01 — Direct action invocation); without `__invoke()`, use a closure that forwards to `dispatch()` instead: `Route::get('/payments/{payment}', fn (Payment $payment) => ShowPaymentAction::make($payment)->dispatch());`

## Rule: Route by owning aggregate identifier only

If a resource (e.g. Payment) has a globally unique identifier and a well-defined ownership (Order), the route MUST be addressed by the resource identifier alone.

Parent identifiers MUST NOT be included in the URL if they can be deterministically derived from the resource itself.

```php
// Correct: payment has globally unique ID
Route::get('/payments/{payment}', ShowPaymentAction::class);

// Incorrect: redundant parent context
Route::get('/orders/{order}/payments/{payment}', ShowPaymentAction::class);
```

**Rationale:** If `Payment` belongs to `Order` and the relationship is stored in the database, there is no need to include `{order}` in the URL. The `{payment}` identifier is sufficient to retrieve both the payment and its owning order.

**Exception:** Use nested routes only when the parent context is mandatory for authorization or when the child resource is not globally addressable.

## Rule: Prefixes represent mandatory context only

Route prefixes MUST be used only when all nested routes share the same mandatory context parameter.

Capability-based or independently addressable resources MUST NOT be grouped under contextual prefixes.

```php
// Correct: all routes require tenant context
Route::prefix('tenants/{tenant}')->group(function () {
    Route::get('/dashboard', TenantDashboardAction::class);
    Route::get('/settings', TenantSettingsAction::class);
});

// Incorrect: payments are globally addressable
Route::prefix('orders/{order}')->group(function () {
    Route::get('/payments/{payment}', ShowPaymentAction::class);
});
```

## Rule: Route parameters are semantic references

Route parameters MUST be named after the domain concept they reference, regardless of how they are resolved internally.

```php
// Correct: semantic naming
Route::get('/orders/{order}', ShowOrderAction::class);
Route::get('/subscriptions/{subscription}', ShowSubscriptionAction::class);

// Incorrect: technical naming
Route::get('/orders/{id}', ShowOrderAction::class);
Route::get('/subscriptions/{uuid}', ShowSubscriptionAction::class);
```

**Rationale:** Route parameter names express domain intent. Internal resolution (ID, UUID, slug) is an implementation detail.

## Rule: Implicit binding is a convenience, not an architecture

Implicit route model binding MAY be used in simple, trusted contexts.

It MUST NOT be used for public, capability-based, or domain-critical flows.

```php
// Acceptable: simple internal admin panel
Route::get('/admin/users/{user}', function (User $user) {
    return view('admin.users.show', ['user' => $user]);
});

// Not acceptable: public capability-based access
Route::get('/invoices/{invoice}/download', function (Invoice $invoice) {
    // Missing: authorization, capability check, audit logging
    return $invoice->downloadPdf();
});
```

**Use implicit binding when:**
- The route is internal or admin-only
- No additional business rules apply
- The resource simply needs to exist

**Avoid implicit binding when:**
- Authorization depends on relationships or capabilities
- Access rules are non-trivial
- Audit trails or logging are required

## Rule: Use implicit binding as existence guard, not as domain resolution

Implicit binding MAY be used when a route only requires a resource to exist and no additional access rules apply.

Implicit binding verifies that a record exists and returns 404 if not. Domain resolution — determining which entity is relevant based on business rules — belongs in Commands or Queries.

```php
// Correct: existence check only
Route::get('/posts/{post}', function (Post $post) {
    return GetPostWithCommentsQuery::make($post->id)->dispatch();
});

// Incorrect: implicit binding used for complex resolution
Route::get('/tenants/{tenant}/active-subscription', function (Tenant $tenant) {
    // Business logic leaking into route layer
    return $tenant->subscriptions()->active()->first();
});
```

## Rule: Route closures may act as micro-controllers

Route closures are acceptable only when they contain a single response statement and no branching logic.

```php
// Correct: single response statement
Route::get('/posts/{post}', fn (Post $post) =>
    GetPostDetailsQuery::make($post->id)->dispatch()
);

// Incorrect: branching logic in closure
Route::get('/posts/{post}', function (Post $post) {
    if ($post->published) {
        return GetPublishedPostQuery::make($post->id)->dispatch();
    } else {
        return GetDraftPostQuery::make($post->id)->dispatch();
    }
});
```

If a route closure requires branching, conditional logic, or multiple statements, use a controller instead.

## Rule: Avoid empty controllers

Controllers MUST exist only when they add HTTP-specific value.

If a route only invokes a single action and returns its result, a route closure is preferred.

```php
// Preferred: no controller needed
Route::get('/posts', fn () => GetPublishedPostsQuery::make()->dispatch());

// Unnecessary controller
class PostController {
    public function index() {
        return GetPublishedPostsQuery::make()->dispatch();
    }
}
```

**Controllers are necessary when:**
- Handling form requests or validation
- Applying authorization policies
- Formatting responses (API resources, Blade views)
- Selecting between multiple actions based on request data
- Setting HTTP headers, status codes, or cookies

## Rule: Route names express business meaning

Route names MUST describe the domain intent, not the technical action performed.

```php
// Correct: business meaning
Route::post('/orders', CreateOrderAction::class)
    ->name('orders.create');

Route::post('/orders/{order}/ship', ShipOrderAction::class)
    ->name('orders.ship');

// Incorrect: technical action
Route::post('/orders', CreateOrderAction::class)
    ->name('orders.store');

Route::post('/orders/{order}/ship', ShipOrderAction::class)
    ->name('orders.update');
```

## Rule: Different user intent means a different route

If two operations represent different user intentions, they MUST be separate routes, even if they modify the same entity.

```php
// Correct: distinct user intentions
Route::post('/posts/{post}/publish', PublishPostAction::class);
Route::post('/posts/{post}/archive', ArchivePostAction::class);
Route::post('/posts/{post}/feature', FeaturePostAction::class);

// Incorrect: single update endpoint for multiple intents
Route::patch('/posts/{post}', UpdatePostAction::class);
```

**Rationale:** Each route represents a use case. Combining multiple intentions into one endpoint obscures business meaning and complicates implementation.

## Rule: Wizards are not single actions

Multi-step wizards MUST NOT be implemented as a single POST endpoint.

Each step that performs a distinct operation MUST have its own route and action.

```php
// Correct: separate routes for wizard steps
Route::post('/onboarding/company', CreateCompanyAction::class);
Route::post('/onboarding/team', InviteTeamMembersAction::class);
Route::post('/onboarding/payment', SetupPaymentMethodAction::class);

// Incorrect: single endpoint handling multiple steps
Route::post('/onboarding', CompleteOnboardingAction::class);
```

**Exception:** A wizard may have a single "review and submit" endpoint if all previous steps have already persisted their data and this final action only commits or publishes the aggregate.

## Rule: POST endpoints map 1:1 to business operations

Each POST route MUST correspond to exactly one business operation.

Avoid generic "update" or "process" endpoints that handle multiple unrelated use cases.

```php
// Correct: one route per operation
Route::post('/invoices/{invoice}/send', SendInvoiceAction::class);
Route::post('/invoices/{invoice}/void', VoidInvoiceAction::class);
Route::post('/invoices/{invoice}/refund', RefundInvoiceAction::class);

// Incorrect: single route handling multiple operations
Route::post('/invoices/{invoice}/process', ProcessInvoiceAction::class);
// (internally branches on 'action' parameter: send, void, refund)
```

**Rationale:** Routes are part of the domain interface. Each route should clearly communicate what operation will be performed.

## Summary

Routing rules enforce:

- Semantic clarity (names reflect domain intent)
- Separation of concerns (transport vs business logic)
- Architectural consistency (routes as use case entry points)
- Predictability (one route = one operation)

Routes are not just URL patterns. They are the domain's public interface.
