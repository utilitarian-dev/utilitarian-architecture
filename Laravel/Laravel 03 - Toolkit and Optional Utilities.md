# Laravel 03 - Toolkit and Optional Utilities

The utilitarian-laravel-toolkit provides a lightweight CQRS implementation for Laravel. It focuses on practical solutions over architectural dogma.

## Philosophy

Events are side effects. Practical over dogmatic.

## Core concepts

The toolkit is built around three operation types:

1. Query - read operations from any source
2. Command - write operations to any source
3. Action - business logic, may orchestrate queries and commands or access data directly

## Buses

- QueryBus executes queries: `QueryBus::execute()`
- CommandBus executes commands: `CommandBus::execute()`
- ActionBus executes actions: `ActionBus::execute()`
- The shortcut to run operations (Query/Command/Action) through their Bus is the method `dispatch()`, so in most cases there is no need to manually instantiate and call the Bus.

## Features

- Query: reading from DB, API, AI, files
- Command: writing to DB, API, files, queues
- Action: logic, decisions, may orchestrate queries/commands or access data directly
- State machine: state management for domain models
- Lightweight DTOs: automatic mapping via cached reflection
- Middleware support for operations

## Middleware

Middleware wraps operation execution to handle cross-cutting concerns. Define middleware in the `middleware()` method.

**Middleware is for infrastructure concerns only.** Acceptable uses: authorization, transactions, logging, caching, timing, audit trails. Not recommended uses: business rule validation, domain logic, conditional workflow branching. If middleware is making business decisions, that logic belongs in the operation itself.

### When to Use Pipeline in Orchestration Actions

- Multiple sequential steps that transform data
- Steps that can be reordered or made conditional
- Complex workflows where each step has clear input/output

### When NOT to Use Pipeline

- Simple two-step operations (just call sequentially)
- When steps don't transform data
- When step order is fixed and simple

For most Actions, direct sequential calls are clearer.


## Dependency Injection

The toolkit uses the `boot()` method for dependency injection. The `execute()` method contains only business logic and is invoked directly, without DI.

| Approach | When to use | Example |
| --- | --- | --- |
| `boot()` method | Dependencies, clients, adapters, configuration | `boot(SearchIndexClient $search): void` |
| Middleware | Cross-cutting infrastructure concerns | `TransactionMiddleware`, `LoggingMiddleware` |

Rules of thumb:

- Use `boot()` for all dependencies needed by the operation.
- If it wraps execution without affecting business logic, use middleware.

## Design principles

- SRP: one class equals one operation
- Explicit buses: always use `QueryBus::execute()` or `CommandBus::execute()`
- Fluent API for configuration before execution
- Composition for complex scenarios
- Laravel-way: do not fight the framework
- Practical: functionality over dogma

## What not to use

- Repositories (covered by Query)
- Services (covered by Action)
- Interfaces everywhere
- DTOs everywhere (only when mixing sources or the logic requires it)

## Optional Instance layer

The toolkit supports an optional Instance layer. It allows an application to customize behavior, routing, views, and bindings without modifying the toolkit or the framework.

**Note:** The Instance layer operates above the CQRS components (Commands, Queries, Actions). It coordinates application-level concerns across multiple features. This is a separate architectural pattern from the core use-case-centric approach. Use it when your application genuinely requires feature coordination at a higher level; do not introduce it just because it exists.

Design principles:

1. Instance is optional. If an application does not define an Instance, nothing happens.
2. Instance extends the application, not the toolkit. Dependency direction is strict:
   - Application -> Toolkit
   - Instance -> Toolkit
3. No default Instance exists. An Instance must be explicitly defined.

How loading works:

The toolkit registers `Utilitarian\Providers\InstanceSupportServiceProvider`. At startup it checks:

```
if (class_exists(\Utilitarian\Instance\InstanceServiceProvider::class)) {
    $this->app->register(
        \Utilitarian\Instance\InstanceServiceProvider::class
    );
}
```

If the class exists, it is registered. If not, nothing happens.

Defining an Instance:

The application provides `Utilitarian\Instance\InstanceServiceProvider`, for example:

```
instance/
└─ src/
   └─ InstanceServiceProvider.php
```

Typical responsibilities:

- Register application-level service providers
- Define or override routes
- Add or override views
- Bind toolkit interfaces
- Coordinate multiple features (web, bot, API, AI)

An Instance should not:

- Contain toolkit logic
- Depend on internal toolkit implementation details
- Assume its own existence
