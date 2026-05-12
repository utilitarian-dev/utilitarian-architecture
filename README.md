# Utilitarian Architecture

A pragmatic approach to organizing Laravel applications around business operations rather than technical layers.

## Overview

Most applications start simple but accumulate friction as business logic grows. Controllers bloat with orchestration. Services appear without clear scope. "Actions" replace services but inherit the same ambiguity.

This architecture separates responsibilities explicitly:

- **Commands** handle writes (create, update, delete)
- **Queries** handle reads (fetch, filter, aggregate)
- **Actions** encapsulate business logic and orchestrate workflows

This separation is enforced by convention. No framework, no runtime dependency. Just clear boundaries.

## Core Principles

**Two-Phase Initialization**
Business parameters are passed via the constructor. Runtime dependencies (database, services) are injected separately before execution. Operations can be constructed, inspected, or queued without triggering infrastructure.

**Soft CQRS**
Read and write separation as a convention, not infrastructure. No separate read database or event sourcing required.

**Business Operations as Units**
Code is organized around what the application does (create order, publish post, generate report) rather than technical layers (controllers, services, repositories).

Actions, Commands, and Queries are operation types, not traditional technical layers. They group executable business behavior while keeping the meaning inside each operation.

**Strong Defaults**
Rules are strong defaults. They should be followed unless a specific project context makes an exception more useful than compliance.

**No Services or Repositories**
Actions replace service orchestration. Queries replace repository reads. Commands replace repository writes. Fewer abstractions, clearer boundaries.

## When This Fits

- Have outgrown basic MVC but don't need enterprise infrastructure
- Have real business workflows beyond simple CRUD
- Domain model is still evolving
- Small to medium teams (2-8 developers)
- Are comfortable with framework conventions (Laravel, specifically)

## When to Look Elsewhere

- Application is genuinely simple (mostly CRUD with minimal business logic)
- Need full DDD with aggregates, bounded contexts, event sourcing
- Require strict architectural enforcement (hard module boundaries, explicit contracts)
- Working outside Laravel ecosystem

## License

This documentation is released under Apache 2.0 License.
