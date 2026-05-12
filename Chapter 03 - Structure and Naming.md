# Chapter 03 - Structure and Naming

This chapter consolidates structure and naming conventions for Laravel applications to keep the codebase navigable and predictable.

## Namespace organization

Group by feature or domain, not by CRUD operation. Keep the structure shallow and consistent across Actions, Commands, and Queries.

Directory example (feature/domain grouping):

```
app/
├── Actions/
│   ├── Auth/
│   │   ├── RegisterUserAction.php
│   │   └── LoginUserAction.php
│   ├── Order/
│   │   ├── CreateOrderAction.php
│   │   └── ProcessPaymentAction.php
│   └── Post/
│       ├── CreateAndPublishPostAction.php
│       └── UpdatePostAction.php
├── Commands/
│   ├── Auth/
│   │   └── CreateUserCommand.php
│   ├── Order/
│   │   ├── CreateOrderCommand.php
│   │   └── UpdateOrderStatusCommand.php
│   └── Post/
│       ├── CreatePostCommand.php
│       └── PublishPostCommand.php
└── Queries/
    ├── Order/
    │   ├── GetOrderQuery.php
    │   └── GetUserOrdersQuery.php
    └── Post/
        ├── GetPostsQuery.php
        └── GetPublishedPostsQuery.php
```

Rules:

- Group by feature or domain, not by CRUD operation.
- Mirror structure across Actions, Commands, and Queries.
- Keep it shallow (max two levels deep). Two levels balance organization with discoverability — deeper nesting increases cognitive load and complicates IDE navigation.

## Cross-domain use cases

When an Action affects multiple domains (for example, `CancelOrderAndCreateRefundAction` touches both Order and Payment), place it where the primary affected aggregate lives — typically the one initiating the workflow.

If the action is equally split, prefer the aggregate that appears first in the name or is the entry point for the user intent.

```
app/
└── Actions/
    └── Order/
        └── CancelOrderAndCreateRefundAction.php  # Lives in Order, affects Payment
```

Avoid creating a "Shared" or "Common" directory for cross-domain actions. This tends to become a dumping ground. Every action has a primary owner.

## Naming conventions

Commands - Verb + Noun + "Command"

Pattern: `{Verb}{Noun}Command`

Examples:

- `CreatePostCommand`
- `UpdateUserCommand`
- `DeleteOrderCommand`
- `PublishPostCommand`
- `ProcessPaymentCommand`

Queries - Get or Find + Noun + "Query"

Pattern: `{Get|Find}{Noun}Query` or `{Get|Find}{Adjective}{Noun}Query`

Examples:

- `GetPostsQuery`
- `GetPostQuery`
- `GetPublishedPostsQuery`
- `FindUserByEmailQuery`
- `GetActiveSubscriptionsQuery`

Rules:

- Use `Get` for fetching collections or known entities.
- Use `Find` for searching or looking up.
- Add adjectives for filtering (Published, Active, Recent).
- Use plural for collections, singular for single entities.

Actions - Verb(s) + Noun + "Action"

Pattern: `{Verb}{Noun}Action` or `{Verb}And{Verb}{Noun}Action` for multi-step workflows

Examples:

- `RegisterUserAction`
- `ProcessOrderAction`
- `CreateAndPublishPostAction`
- `CancelAndRefundOrderAction`

Rules:

- Minimum one verb describing the primary operation.
- Use "And" to combine multiple operations when the action orchestrates distinct steps.
- Actions can be more descriptive than commands.

## Directory naming: singular by default

A directory is a space of responsibility, not a data collection. Use singular form to represent the concept or bounded context.

Inside `Order/` you may have:
- `FindOrder.php` — fetches one
- `ListOrders.php` — fetches many
- `SyncOrders.php` — batch operation

Data quantity varies, the concept stays the same.

Prefer:

- `Order/`
- `User/`
- `Payment/`

Avoid:

- `Orders/` — implies a collection, creates ambiguity
- `Users/`
- `Payments/`

Exception: use plural when the directory is inherently a collection, such as `Reports/` or `Migrations/`.

## Infrastructure outside, business inside

When an infrastructure adapter is involved, the data source is the leading context — place it at the top level, then group by domain.

Structure: `{DataSource}/{DomainEntity}/{UseCase}`

Example:

```
app/
└─ Airtable/
   ├─ Order/
   │  ├─ FindOrderById.php
   │  ├─ ListOrders.php
   │  └─ OrderRecordMapper.php
   ├─ Customer/
   └─ Product/
```

This keeps the infrastructure adapter explicit and the domain structure scalable.

Avoid:

- `AirtableOrder` or `AirtableOrders` (flat, not scalable)
- `Order/Airtable` (makes infrastructure part of the domain)
- `Orders/Airtable` (plural confusion + wrong hierarchy)

Note: If you find yourself with multiple infrastructure sources (`Airtable/`, `Salesforce/`, `Api/`), consider whether you are creating an implicit adapter layer. This may be appropriate, but be aware that it introduces a structural pattern beyond basic CQRS. Keep it simple unless the complexity is justified.
