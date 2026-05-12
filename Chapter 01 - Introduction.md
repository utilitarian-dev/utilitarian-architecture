# Chapter 01 - Introduction

Utilitarian Architecture is a pragmatic approach to building applications that balances code quality, developer productivity, and maintainability.

Architectural patterns are treated as tools, not goals. Abstractions are introduced only when they reduce cognitive load or make the system easier to change. Solving real business problems efficiently matters more than maintaining perfect layer boundaries.


## The Problem

Framework-provided structures work well for simple applications, but friction increases as business logic grows. Controllers accumulate orchestration and business rules. Supporting classes proliferate without clear responsibilities. "Services" and "Repositories" appear with inconsistent scope: some encapsulate a single operation, others grow into god objects. Actions could provide good separation and simplicity, but they become difficult to navigate: should this be one action or several? What to call it? Where to put it?  Code placement becomes uncertain.

Utilitarian Architecture addresses this by providing:

- A clear organizational principle: the use case as the core unit
- Three well-defined component types with explicit responsibilities
- Minimal infrastructure overhead
- Naming conventions that reveal intent


## Core Principles

**Use case as the core unit.** A use case is a complete business operation. The architecture is organized around these operations, not around technical layers.

Actions, Commands, and Queries are not technical layers in the traditional layered architecture sense. They are executable units of business behavior. The directory structure groups operation types for discoverability; the business meaning still lives inside each operation.

**Soft CQRS.** Reads and writes are intentionally separated, but not as a full CQRS system. Commands change state. Queries read data. Actions contain business logic and may orchestrate workflows.

**Pragmatism over dogma.** The architecture favors clarity and usefulness over theoretical purity. Abstractions exist only when they reduce cognitive load or simplify change.

**Strong defaults.** Rules in this architecture are strong defaults. They should be followed unless a specific project context makes an exception more useful than compliance. Exceptions should be intentional, local, and easy to explain.

**Freedom of tooling.** Inside a command, query, or action, the implementation can use whatever works best: Eloquent, Query Builder, raw SQL, HTTP clients, SDKs, or files. This freedom applies to *how* you implement the operation, not to *what* it does. Commands must still write. Queries must still read.

**Tools are optional.** Value objects, DTOs, state machines, and custom helpers are used when they add real value, not because the architecture demands them.


## Who This Is For

- Teams that need structure beyond basic MVC but want to avoid heavy frameworks
- Projects with real business workflows and growing complexity
- Developers who want clarity without excess formalism
- Small-to-medium teams (2-8 developers) comfortable with conventions
- Growing applications that started simple and need structure


## Who This Is Not For

- Systems that require full domain-driven design or event sourcing
- Projects that can stay small and simple with plain MVC
- Teams that need strict, enterprise-grade governance or hard boundaries between modules
