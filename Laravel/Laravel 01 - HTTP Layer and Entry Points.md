# Laravel 01 - HTTP Layer and Entry Points

In Laravel applications, the HTTP layer offers several permissible entry points: built-in routing methods, controllers, and route closures.

The choice depends on:

- Complexity of the scenario
- The need to interact with HTTP context
- How strictly business logic is isolated from transport concerns

A foundational rule remains consistent:

The HTTP layer is only responsible for receiving a request and returning a response. Business logic must live outside of it.

## Static content and navigation

`Route::view()`, `Route::redirect()`, `return view()`

These approaches are suitable only for pages that have:

- No user input handling
- No database interactions
- No conditional or computational logic

Use cases: landing pages, "About Us", privacy policy, terms of use.

Key benefits:

- No meaningless controllers
- Minimal boilerplate
- Reduced cognitive load
- Clean, readable route definitions

## Controllers

A controller acts as a transport coordination mechanism. It should not contain business logic.

A controller is appropriate when:

- Orchestrating HTTP request and response (form requests, authorization, status codes, headers, cookies)
- Transforming data (Blade views, API resources, DTO formatting)
- Selecting execution flows based on input (conditional branching based on request content)
- Accessing data added by middleware

Controllers always delegate business logic to actions, commands, or other application-level objects.

### Controller responsibility boundary

A controller may decide *which* Action to call based on request data — this is transport coordination. A controller must not decide *how* the Action should behave internally — that is business logic.

```php
// Acceptable: choosing which action to call
public function store(Request $request)
{
    if ($request->boolean('publish')) {
        $action = CreateAndPublishPostAction::make(...);
    } else {
        $action = CreatePostAction::make(...);
    }
    return $action->dispatch();
}

// Not acceptable: business logic in controller
public function store(Request $request)
{
    $discount = $request->user()->isPremium() ? 0.2 : 0;  // Business rule
    $finalPrice = $this->calculatePrice($request->items, $discount);  // Business logic
    // ...
}
```

If the controller needs to compute values, apply business rules, or make domain-level decisions, that logic belongs in an Action.

## Request-scoped context

Middleware often resolves request-scoped data that Actions need: the current tenant, active locale, resolved shop, or any parameter derived from the request that a route group shares. Actions must not access this data through global helpers, static facades, or service locators.

The correct transport is `$request->attributes`, a Symfony `ParameterBag` designed for sharing data between middleware and controllers within a single request lifecycle.

```php
// Middleware — resolves and stores context
public function handle(Request $request, Closure $next): Response
{
    $scope = (new ResolveShopScopeQuery($request))->execute();
    $request->attributes->set(ShopScope::class, $scope);

    return $next($request);
}

// Controller — reads attributes, passes explicitly to Action
public function show(string $slug, Request $request): Response
{
    $scope = $request->attributes->get(ShopScope::class);

    return (new ShowProductAction($slug, $scope))->execute();
}

// Action — receives scope as a plain constructor parameter
public function __construct(
    public readonly string    $slug,
    public readonly ShopScope $scope,
) {}
```

Using the class name as the key (`ShopScope::class`) prevents string-key collisions and makes the dependency explicit.

**Not acceptable — misuse of the logging infrastructure:**

```php
// Middleware
Context::addHidden(ShopScope::class, $scope);

// Controller
$scope = Context::getHidden(ShopScope::class);
```

`Context::hidden()` is Laravel's log-enrichment facility. "Hidden" means hidden from log output, not a private store for dependencies. Using it as a request-scoped container couples the application layer to a logging concern.

**Not acceptable — global helpers:**

```php
$id     = currentShopId();   // hidden coupling, untestable
$locale = activeLocale();    // bypasses the explicit data flow
```

### Naming

Name these DTO classes to reflect their domain, not their transport mechanism. Avoid the suffix `Context` — it creates ambiguity with Laravel's `Context` facade.

| Recommended | Avoid |
| --- | --- |
| `ShopScope` | `ShopContext` |
| `TenantScope` | `TenantContext` |
| `LocaleScope` | `LocaleContext` |

### When middleware is not needed

If only one or two routes need the resolved data, resolving directly in the controller is cleaner:

```php
public function show(string $slug, Request $request, ResolveShopScopeQuery $query): Response
{
    $scope = $query->execute($request);

    return (new ShowProductAction($slug, $scope))->execute();
}
```

Use middleware when the scope is required across many routes and resolving it once per request matters.

## Closures

Closures inside routes are acceptable in two distinct roles and should not be mixed.

### Closures with inline logic (limited use)

Valid only when:

- Logic is trivial and flat
- No branching or growing complexity expected
- The code will remain permanently small

Example: health checks, simple redirects, temporary system endpoints.

### Closures as entry points (recommended)

A closure may serve solely as a lightweight endpoint entry. It delegates execution entirely to the application layer.

In this case:

- The closure contains zero business rules
- It acts similarly to a thin controller
- The action owns the use case execution



## Direct action invocation

Route to action is acceptable when a use case is:

- Single-step
- Requires minimal HTTP processing
- Operates solely on route parameters and returns a result

Benefits:

- Minimal infrastructure code
- Highly testable
- Business logic isolated from HTTP
- Reuse from CLI, jobs, and tests

## Entry point selection guidance

| Scenario | Recommended entry point |
| --- | --- |
| Static page | `Route::view` |
| Redirect or navigation | `Route::redirect` |
| HTTP orchestration and formatting | Controller |
| Atomic use case | Closure to Action |
| Temporary trivial logic | Inline closure (explicitly justified) |

## Foundational principle

Closures and controllers are transport adapters, not business containers. Action objects are where the business meaning lives.

Invokable syntax is optional, not a requirement.

For routing rules and URL design principles, see Laravel 02 - Routing Rules.
