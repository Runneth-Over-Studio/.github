# Runneth Over Studio Development Conventions

## Purpose

This document defines the current development conventions intended for
contributors of Runneth Over Studio repos.

These conventions are defaults, not dogma. They should promote clarity,
consistency, maintainability, separation of concerns, and simplicity.
Reconsider a convention when a concrete design problem demonstrates that
another approach better serves the project.

Top-level product or architectural principles take precedence where a
lower-level convention conflicts.

------------------------------------------------------------------------

## 1. General Design Philosophy

### Prefer Simple Designs

Follow KISS. Do not introduce abstractions, layers, patterns, or object
types solely to satisfy an architectural diagram. Every abstraction
should solve a concrete problem or protect an intentional boundary.

Avoid speculative architecture for capabilities projects do not yet
require.

### Enforce Separation of Concerns

Code should live with the responsibility it implements.

-   Presentation does not implement business logic.
-   Business does not implement persistence concerns.
-   Data access does not implement business validation.
-   External integration details do not leak into core business
    concepts.
-   Persistence structure does not define business structure.
-   A project reference grants technical access, not architectural
    permission.

Only protocols, services, DTOs, and other types intentionally exposed
across a boundary should be consumed by another tier or capability.

### Preserve Locality of Behaviour

Within Business, organize code so behavior related to a capability is
easy to discover and reason about locally. Prefer cohesive
feature/domain organization over scattering related behavior throughout
large technical folders.

Reference: [Locality of
Behaviour](https://htmx.org/essays/locality-of-behaviour/)

### Prefer Modularity

Solutions should be modular both horizontally through architectural
boundaries and vertically through cohesive business capabilities.

Modules should expose deliberate contracts rather than internal
implementation details.

------------------------------------------------------------------------

## 2. .NET Platform

Projects should target the current Long-Term Support version of .NET,
following the even-numbered .NET release cadence.

Prefer the highest supported even-numbered LTS release unless a concrete
compatibility constraint requires otherwise. Do not upgrade merely to
consume a non-LTS release without a compelling reason.

------------------------------------------------------------------------

## 3. Solution Architecture

Typically solutions should follow a Data, Business, Presentation
architecture, extended where responsibilities do not naturally belong
to those three tiers.

``` text
MyApp.Data
    Persists and retrieves application-owned state.

MyApp.Business
    Defines business concepts, application behavior, validation,
    protocols, and intentionally exposed contracts.

MyApp.Integrations
    Adapts external applications, standards, and services
    into capabilities understood by the application.

MyApp.Presentation.*
    Provides interfaces through which users interact with the
    application. Also serves as the composition root, configuring DI at startup.

            services.AddBusinessServices();
            services.AddDataServices();
            services.AddIntegrationServices();
```

### Data

`*.Data` owns persistence concerns for application-owned information,
including database access, persistent/data entities, schema and
migrations, persistence-specific mapping, repository/store implementations, local
caches and indexes, and synchronization persistence state where
appropriate.

Data access must not implement business validation or application
decisions. Persistence implementation details should not leak into
Business or Presentation.

### Business

`*.Business` owns application and domain behavior, including
business concepts, processes/services, validation, application DTOs,
provider protocols/contracts, semantic relationships, and expected
application failure states.

Business should not define persistence-specific entities or depend on
persistence implementations. Business defines the persistence capabilities
it requires through application-facing contracts; Data implements those
contracts. Business should remain independent of presentation technology.

### Integrations

External systems are capabilities, not merely data stores. Rather than
being forced into `*.Data`, provider implementations should generally
live in an integration project organized by provider-specific namespaces
or folders. Dedicated integration projects may be used when an
integration's size, dependencies, or isolation requirements justify them.

Integration implementations:

-   Understand the external provider API, protocol, and data formats.
-   Implement capability contracts.
-   Translate provider-specific representations into application-defined
    contracts and DTOs.
-   Keep provider-specific types from leaking into `*.Business`.
-   Do not become canonical owners of external source data.

### Presentation

There may be multiple presentations over the same business
functionality.

``` text
MyApp.Presentation.Desktop
MyApp.Presentation.Web
MyApp.Presentation.Mobile
MyApp.Presentation.Cli
```

Presentation projects do not implement business or persistence logic.
Presentation-specific state and binding models belong in the appropriate
Presentation project.

------------------------------------------------------------------------

## 4. Project Naming

Use the genericized application name for the solution and as the root in
project names, with a suffix describing the architectural tier,
capability, or presentation target.

Examples:

``` text
MyApp/
├── src/
    ├── MyApp.slnx
    ├── MyApp.Data/
    ├── MyApp.Business/
    ├── MyApp.Integrations/
    ├── MyApp.Presentation.Desktop/
    └── MyApp.Presentation.Web/
```

For presentation projects, include the target platform/presentation in
the suffix.

------------------------------------------------------------------------

## 5. Presentation Patterns

### Desktop and Mobile GUI

Desktop and mobile GUI applications should generally follow MVVM where
appropriate.

``` text
View
    Visual structure and presentation behavior.

ViewModel
    Presentation state, commands, and coordination.

Model
    Presentation-facing data suitable for binding.
```

Business rules remain in Business.

### Web

Use a pattern appropriate to the chosen web technology while maintaining
the same separation-of-concerns rules. Do not force desktop MVVM
concepts into a web framework where they do not naturally fit.

### CLI

MVVM is not required for CLI applications. Prefer the simplest pattern
suited to command-line interaction, for example:

``` text
Command
    ↓
Business service/process
    ↓
Result
    ↓
Console rendering
```

The absence of MVVM does not relax architectural separation.

------------------------------------------------------------------------

## 6. Pattern Naming Conventions

Use architectural suffixes only when a class actually performs that
role.

-   View classes end in `View`.
-   Presenter classes end in `Presenter`.
-   ViewModel classes end in `ViewModel`.
-   Controller classes end in `Controller`.
-   Classes that do not perform these roles must not use these suffixes.

------------------------------------------------------------------------

## 7. Business Object Vocabulary

### Persistent Entities / Data Entities

A **persistent entity** or **data entity** is a uniquely identifiable
persisted data representation. Use these qualified terms in
architectural discussion when ambiguity is possible.

Do not call every business object an entity merely because it has
identity.

Persistence structure does not define business structure. Multiple data
entities may support one business concept, and a data entity may exist
only for caching, synchronization, indexing, auditing, integration
state, or other infrastructure bookkeeping.

Persistent entities should generally remain simple data-oriented
objects.

### DTOs

DTOs are lightweight objects intended to transfer data across an
intentional boundary.

DTOs should:

-   Be serializable by design.
-   Be POCOs.
-   Generally be C# records.
-   Contain data rather than complex behavior.
-   Represent the contract the application intends to expose rather than blindly
    mirror persistence or provider representations.

Example:

``` csharp
public sealed record CreateGoalDto(
    string Name,
    string? Description,
    Guid LifeDomainId);
```

External integrations may have provider-specific DTOs in addition to
application-facing DTOs. Do not expose an external provider DTO directly
merely to avoid writing a mapping.

### Business Concepts / Domain Objects

A **business concept**, sometimes referred to as a **domain object**, represents a
real-world or application-specific concept required by business logic. It is not defined
by whether or how it persists.

A business representation may have one persistent representation,
multiple persistent representations, or none.

Business objects should generally remain straightforward data-oriented
representations. Prefer focused business services/processes for
non-trivial behavior rather than automatically placing complex behavior
on the objects themselves.

Useful DDD ideas may be adopted when they solve a concrete problem, but
we do not require DDD terminology or rich domain objects by default.

### Models

A **Model** is a representation intended for presentation consumption.
Models may represent a view of business concepts, DTOs, or composed
information from multiple sources.

Models represent data in a shape appropriate for presentation consumption.
Presentation Models belong to Presentation when their shape exists
specifically for UI needs.

### Do Not Manufacture One-to-One Representations

Do not automatically create all of the following for every concept:

``` text
GoalEntity
Goal
GoalDto
GoalModel
```

Create a distinct representation only when a real boundary or
transformation requires it. KISS takes precedence over architectural
ceremony.

------------------------------------------------------------------------

## 8. Business Behavior

### Prefer Focused Processes and Services

Generally favor restrained behavior on data-oriented objects.
Non-trivial operations should normally live in focused
services/processes near the capability they implement.

For example:

``` text
Goals/
├── Goal.cs
├── Objective.cs
├── CreateGoalProcess.cs
├── AddObjectiveProcess.cs
└── CompleteGoalProcess.cs
```

Exact organization should follow the capability and Locality of
Behaviour rather than a mandatory folder template.

### Validation

Application DTOs should be validated at the appropriate
business/application boundary.

Do not put business validation in Data, depend on UI validation as sole
enforcement, or treat database constraints as a replacement for business
validation. Database constraints may still enforce data integrity
defensively.

------------------------------------------------------------------------

## 9. Integration Contracts

Business defines capabilities; integration projects implement providers.

Prefer capability-focused contracts such as:

``` text
IKnowledgeProvider
IProjectProvider
ISourceControlProvider
ICalendarProvider
IFileProvider
```

over a kitchen-sink `IIntegrationProvider`.

One provider may implement multiple capabilities. Provider-specific
types remain inside the corresponding integration project. Application-facing
contracts use application-defined DTOs and business concepts.

Read and write authority should be distinguishable, particularly where
agents or automation may exercise capabilities.

------------------------------------------------------------------------

## 10. Failure Modeling and Exceptions

Expected operational outcomes should generally be represented explicitly
rather than communicated through exceptions. When an operation has
multiple anticipated outcomes that callers are expected to handle, prefer
an appropriate result type, discriminated result, Try pattern, or other
explicit representation.

Exceptions play a supporting role. Use them for genuinely exceptional
conditions, programming/contract violations where appropriate, and
unexpected failures that cannot reasonably be represented as part of the
operation's normal result contract. Do not use exceptions as ordinary
control flow.

``` text
Operation outcome
│
├── Expected outcome
│   ├── Success
│   ├── NotFound
│   ├── Unauthorized
│   ├── Forbidden
│   ├── Unavailable
│   └── ValidationFailure
│
│       → explicit state / Result type
│
└── Exceptional failure
    ├── violated programming invariant
    ├── unexpected implementation failure
    └── condition the current abstraction cannot reasonably model/recover from

        → exception
```

Do not catch an unexpected exception merely to convert every possible
failure into a generic failure Result. Doing so destroys one of the main
benefits of exceptions: unexpected failures propagate with diagnostic
information and stack traces.

🔴 Destroys diagnostic information:
``` csharp
catch (Exception ex)
{
    return Result.Failure(ex.Message);
}
```

🔴 Hides bug:
``` csharp
catch (NullReferenceException)
{
    return KnowledgeResult.InvalidResponse();
}
```

🟢 Appropriate at the integration boundary, because we're deliberately
translating a provider-specific failure into an anticipated application
state:
``` csharp
catch (JoplinAuthenticationException)
{
    return KnowledgeResult.Unauthorized();
}
```

| Condition                                                       | Preferred mechanism                            |
| --------------------------------------------------------------- | ---------------------------------------------- |
| Expected business/operational outcome                           | Explicit state / `Result<T>`                   |
| Caller/programmer violates API contract                         | Standard exception where appropriate           |
| Unexpected implementation/system failure                        | Exception                                      |
| Provider exception representing an expected application outcome | Catch at boundary and translate to Result      |
| Unknown/unexpected provider exception                           | Allow to propagate or wrap with useful context |

------------------------------------------------------------------------

## 11. C# Coding Conventions

### Namespaces

Prefer file-scoped namespaces.

``` csharp
namespace MyApp.Business.Goals;
```

### Local Variable Declarations

Prefer an explicit type with target-typed construction:

``` csharp
Goal goal = new();
```

over:

``` csharp
var goal = new Goal();
```

Use judgment where explicit typing would make code substantially less
readable.

### Constants

Name constants using uppercase snake case.

``` csharp
private const string API_ENDPOINT = "https://rest.example.com/v1";
```

### Private Fields

Use camel case with a leading underscore.

``` csharp
private int _counter;
```

### Public and Internal Members

Use PascalCase.

``` csharp
public required DateTime TimeStamp { get; set; }
```

### General C# Style

Unless the application defines a more specific convention, generally follow
[Microsoft C# Coding
Conventions](https://learn.microsoft.com/dotnet/csharp/fundamentals/coding-style/coding-conventions).

### Library Design

Libraries should generally follow the [.NET Framework Design
Guidelines](https://learn.microsoft.com/dotnet/standard/design-guidelines/)
where applicable.

------------------------------------------------------------------------

## 12. Dependency and Boundary Rules

The conceptual dependency relationship is:

``` text
          Presentation.*
                │
                ▼
             Business
          ▲           ▲
          │           │
        Data     Integrations
```

Business is the architectural center, while the executable Presentation
project is the composition root handling DI registration.

Dependencies across infrastructure boundaries should follow the Dependency
Inversion Principle. Business defines application-facing contracts for
persistence and external capabilities; Data and Integrations reference
Business and implement those contracts. Business services depend on the
contracts rather than their implementations. The Presentation composition
root registers concrete implementations with dependency injection.

``` text
             Presentation.Desktop
             /        |         \
            ▼         ▼          ▼
       Business     Data     Integrations
          ▲           │          │
          └───────────┴──────────┘
```

So a practical project graph is:

``` text
MyApp.Presentation.Desktop
├── MyApp.Business
├── MyApp.Data
└── MyApp.Integrations

MyApp.Data
└── MyApp.Business

MyApp.Integrations
└── MyApp.Business

MyApp.Business
└── no application-project dependencies
```

Exact project-reference mechanics may vary as contracts are factored,
but preserve these rules:

-   Business does not depend on Presentation.
-   Business does not depend on provider-specific integrations.
-   Business does not depend on persistence implementation details.
-   Data implements persistence responsibilities required by Business.
-   Integrations implement external capabilities required by Business.
-   Presentation consumes intentionally exposed Business capabilities.
-   Provider-specific DTOs remain inside their integration boundary.
-   Persistent/data entities remain inside the persistence boundary
    unless deliberately exposed for a specific reason.

Technical visibility does not grant architectural permission to consume
an implementation type.

------------------------------------------------------------------------

## 13. Coding Guidance

When generating or modifying code:

1.  Preserve Data, Business, Integration, and Presentation boundaries.
2.  Do not introduce provider-specific types into Business.
3.  Do not introduce persistent/data entities into Presentation.
4.  Do not implement business validation in Data.
5.  Do not implement business logic in Presentation.
6.  Prefer focused capability-local services/processes over large
    generic managers.
7.  Prefer simple data-oriented business objects unless object behavior
    provides a clear advantage.
8.  Use records for DTOs unless there is a concrete reason not to.
9.  Do not create DTOs, Models, data entities, interfaces, repositories,
    or mappings merely for architectural symmetry.
10. Prefer explicit contracts at project boundaries.
11. Preserve external systems as canonical owners of information that the
application does not intrinsically own.
12. Prefer open protocols and provider abstractions over vendor
    coupling.
13. Follow this document's C# naming and formatting conventions.
14. Target the current even-numbered .NET LTS release unless the
    solution explicitly specifies otherwise.
15. Ask whether an abstraction solves a current problem before adding
    it.
16. If a convention appears inappropriate for a concrete problem,
    explain the tradeoff rather than silently working around it.

------------------------------------------------------------------------

## 14. Decision Heuristics

When deciding where new code belongs:

**Is this about storing application-owned information?**\
It probably belongs in `*.Data`.

**Is this an application rule, concept, process, validation, or capability
contract?**\
It probably belongs in `*.Business`.

**Does it understand an external application's API, protocol, or
representation?**\
It probably belongs in `*.Integrations/<Provider>`.

**Does its shape or behavior exist specifically because of a user
interface?**\
It probably belongs in `*.Presentation.<Platform>`.

**Does a new class merely duplicate another representation without
protecting a boundary?**\
Do not create it yet.

**Does a database table exist only for infrastructure reasons?**\
Do not invent a matching business concept.

**Does a business concept persist?**\
That does not make the persistence representation the business model.

------------------------------------------------------------------------

## 15. Guiding Architectural Rule

> **Architecture should make intended behavior easy to find, intended
> dependencies obvious, and unintended coupling difficult.**

Favor explicit boundaries and consistent conventions, but simplicity and
clear responsibility take precedence over pattern compliance for its own
sake.
