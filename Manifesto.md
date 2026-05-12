# Utilitarian Architecture Manifesto

## What It Is

Utilitarian Architecture is a pragmatic approach to building applications (primarily in Laravel), that prioritizes usefulness, clarity, and real-world outcomes over theoretical purity.

Architectural patterns are treated as tools, not goals. Abstractions are introduced only when they reduce cognitive load, improve readability, or make the system easier to change.


## Who It’s For

Utilitarian Architecture is well suited for projects where:

- The project has outgrown basic MVC but doesn't need heavy architectural overhead.
- The system has complex business logic but only a few business domains.
- The domain model evolves alongside the product, making upfront design impractical.
- Clear boundaries and conventions are needed without heavy coordination overhead.
- Shipping matters more than architectural debate.
- The team is small (or solo).


## What Problem It Solves

Framework-provided project structures work well for simple applications, but friction tends to increase as business logic grows:

- Entry-point layers (such as controllers) accumulate orchestration and business rules.
- Supporting classes proliferate without clear responsibilities or boundaries.
- Code placement becomes inconsistent, leading to recurring structural uncertainty.

Utilitarian Architecture introduces clear, repeatable answers to these problems while remaining lightweight and adaptable across frameworks. Laravel is used as a primary reference point, not a limitation.


## Core Principles

The core unit of the system is the business operation (use case). The architecture is organized around expressing, executing, and maintaining these operations.

Read and write responsibilities are separated as a convention, not a dogma.
Commands perform state changes.
Queries retrieve data. Repeated access to the same read-only data within a single execution may be handled via request-scoped memoization at the Query level. Such memoization is an execution-level optimization and must not alter observable behavior or be used for data that may change during execution.

Actions represent application-level behavior and are the primary unit of application logic.

They encapsulate calculations, transformations, business rules, and policy enforcement. When a business operation requires coordination, Actions may orchestrate Commands, Queries, and other Actions. The initiating Action owns the overall control flow.

Commands, Queries, and Actions follow Two-Phase Initialization.
Business parameters are provided via the constructor and remain immutable,
Runtime dependencies are injected during an optional boot() phase, executed once by the Bus before execution.
The execute() method contains only business logic and is invoked directly, without dependency injection.

Services and Repositories are not part of this architecture. Their responsibilities are covered by Actions (orchestration, previously "services") and Queries/Commands.

Infrastructure clients, framework primitives, SDKs, and adapters may still be used as implementation details inside operations. They do not form a separate application service or repository layer.

Any architectural element that doesn't make the system simpler or easier to change is not worth adding.


## What It’s Not

- It is not a framework or toolkit. Utilitarian Architecture is a set of conventions and decision guidelines, not a runtime dependency.
- It does not require comprehensive domain modeling, event sourcing, or upfront architectural completeness.
- It does not impose rigid structural layers or containerized application schemas.
- It is not a universal solution. Some projects are simple enough that this level of structure is unnecessary, but it can be applied early to keep feature development simple as the product grows.
