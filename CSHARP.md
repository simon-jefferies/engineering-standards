# C# engineering standards

Use this guide when designing, implementing or reviewing C# applications and libraries. It is the maintained source for the shared object-design and C# principles in this repository. Apply the sections relevant to the change, together with the consuming project's requirements. For WPF work, also read [WPF.md](WPF.md).

These are engineering standards for people and coding agents. The linked Microsoft documentation explains language and runtime behaviour; project decisions still require judgement.

Last reviewed: 19 October 2026.

## Core expectation

Deliver working behaviour with coherent responsibilities, encapsulated state, explicit dependencies and deliberate resource ownership. Treat these qualities as completion criteria throughout the work. Explain material departures and their consequences.

Object-oriented design, SOLID principles, meaningful naming and the applicable design-pattern contracts are required standards. Apply them to the responsibilities and relationships affected by the change. Resolve material violations within the agreed scope before reporting completion. Explain a conflict with an explicit project requirement and record the resulting exception; describing a violation does not by itself satisfy the standard.

Choose the smallest coherent design that meets the requirement. Class counts, interface counts and line counts do not establish design quality. Address material design weaknesses within the agreed scope; keep unrelated restructuring separate.

## Before changing code

- Establish the expected behaviour, relevant constraints and evidence for the change. Distinguish observed implementation from intended design, and observed facts from assumptions.
- Check target frameworks, supported operating systems, SDK, language version, package versions, build commands and test facilities. Use syntax and APIs supported by the project, including .NET Framework where applicable.
- Follow repository instructions, `.editorconfig` and established analyser settings. Keep language and build settings reproducible; do not switch to `latest` or preview features merely to make a new example compile.
- Identify affected public contracts, consumers, serialized data and compatibility requirements before changing them. Treat package or toolchain upgrades as deliberate changes with a concrete purpose.

References: [language version configuration][language-version], [library design guidance][library-design].

## Object design and SOLID

Put behaviour with the object responsible for the concept. Keep business decisions, application coordination and external integration distinguishable, and separate them where their responsibilities require it. No fixed number of layers, projects or classes is required.

Review the changed design against the applicable SOLID principles:

| Principle | Check in the resulting design |
| --- | --- |
| Single responsibility | The type has a coherent purpose. Unrelated business rules, persistence, UI work and orchestration have not accumulated in one manager, service or ViewModel. |
| Open/closed | Known variation has a useful extension point where needed. Do not build speculative plugin systems or replace every small conditional with a hierarchy. |
| Liskov substitution | An implementation or derived type honours the contract its callers rely on, including valid inputs, outcomes and invariants. |
| Interface segregation | Consumers depend on focused roles. An interface need not reproduce every public member of its implementation. |
| Dependency inversion | Business policy can be understood and exercised independently of a particular UI, database or external service. Contracts belong at the boundary that needs them. |

- Protect business invariants through meaningful operations. Callers should not need to coordinate arbitrary setters or mutate an exposed collection to perform one business action.
- Keep internal representation private and use the narrowest useful accessibility. Provide observable outcomes and operations that describe intent.
- Prefer composition over inheritance. Use inheritance for a genuine substitutable relationship when it serves the design better than collaborating objects; keep base classes focused and extension points deliberate.
- Framework-required inheritance and focused framework bases remain valid. Follow their contracts while composing unrelated application behaviours outside the hierarchy.
- Introduce a type or interface for a useful concept, contract, substitution or independently changing responsibility. Avoid one interface per class, unstructured helper collections and layers that only forward calls without serving a boundary.
- Use entities and value objects where identity, invariants or domain operations matter. DTOs, request/response objects, projections and editable presentation models can legitimately carry data without rich domain behaviour.
- Keep constructors focused on establishing valid initial state and dependencies. Avoid hidden I/O, calls to overridable members or untracked asynchronous work during construction. Use explicit operations or factories when creation needs additional work.

References: [object-oriented C#][object-design], [library design guidance][library-design].

### Design patterns

Choose established patterns to solve the actual design problem and preserve their intended contracts. Make the participating objects, dependency direction and lifetime understandable. A pattern name does not replace demonstrating why the design fits.

| Need | Pattern to consider |
| --- | --- |
| Interchangeable behaviour with a stable consumer contract | Strategy, composed into the consumer. |
| Translate an external API into an application contract | Adapter. |
| Encapsulate meaningful construction or implementation selection | Factory. |
| Add a distinct responsibility around an existing contract | Decorator. |
| Notify independent participants about an event | Observer, with explicit subscription ownership. |

Use the relevant pattern consistently where its collaboration is needed. Keep simple operations direct when no such variation or boundary exists; do not introduce an abstraction solely to attach a pattern name to it.

## Contracts, state and types

- Make nullability, allowed values, results, failure behaviour and ownership clear at boundaries. Prefer meaningful names and types over ambiguous Boolean switches or loosely related argument lists.
- Enable nullable reference analysis where supported and resolve warnings in the code being changed. Plan broader adoption in established projects. Nullable annotations are compile-time analysis, not runtime validation; validate untrusted input and enforce essential public contracts.
- Use `null` to represent absence only where that meaning is explicit. Avoid arbitrary empty objects, default values or null-forgiving operators that conceal an invalid state. Document a justified suppression at the relevant boundary.
- Choose classes, records and structs according to identity, equality, copying and mutability requirements. Use records when their value-equality semantics fit. Avoid mutable structs where copying would make state changes surprising.
- Treat `init`, `readonly` references and records as limited guarantees. They do not make every referenced object immutable. Protect mutable children and collections when the owning object's invariants require it.
- Expose collection capabilities that callers need. A read-only interface restricts that interface's operations; it does not establish a snapshot, immutable elements or exclusive ownership of the underlying data. Choose a wrapper, snapshot or immutable representation deliberately.
- Keep equality and hash codes consistent. Avoid changing equality-relevant state while an object is used as a dictionary key or set member.
- Review source, binary, behavioural and serialization compatibility when changing a public library contract. Renaming a public member or changing a default can affect consumers even when the local solution still builds.

References: [nullable reference types][nullable], [records][records], [read-only collections][readonly-collection], [library design guidance][library-design].

## Dependencies and composition

- Supply required collaborators explicitly, usually through constructors. A long dependency list is a prompt to review responsibility, not to hide dependencies inside a container or service locator.
- Construct the object graph at an appropriate composition boundary. Keep container resolution and registration out of business decisions. Dependency injection does not require using a container.
- Prefer direct collaboration for a direct operation. Use events or messaging where independent participants need notification, with an explicit contract and ownership. Do not introduce another event bus, mediator or generic repository solely to satisfy a pattern.
- Keep domain and reusable application logic independent of UI objects, storage details and transport-specific types. Adapt external data at the boundary where the translation belongs.
- Static methods are suitable for stateless operations with explicit inputs. Global mutable state and hidden access to configuration, time, I/O or shared services make dependencies harder to reason about and control.
- Use existing library and framework capabilities where they fit. Add a package when it solves a concrete need and its compatibility and maintenance cost are justified.

Reference: [dependency injection guidelines][di].

## Asynchronous work and concurrency

- Return `Task` or `Task<T>` for ordinary asynchronous operations and await them through the call chain. Reserve `async void` for framework-required event handlers, with failure handling at that boundary. Give background work an owner that observes completion and exceptions.
- Use asynchronous I/O APIs directly. Use `Task.Run` for suitable CPU work or identified blocking work that can safely execute elsewhere. Adding `async` does not itself move work to another thread.
- Avoid `.Wait()`, `.Result` and other synchronous waits in asynchronous flows, particularly on UI or request-handling paths. Do not wrap synchronous blocking in an async signature and assume it becomes safe.
- Accept and propagate cancellation where the operation supports it. Cancellation is cooperative: define when it can be honoured, clean up owned work and preserve valid state. Do not report a completed operation as cancelled merely because a token was cancelled afterwards.
- Follow the application's context requirements and the library's policy for `ConfigureAwait`. Code that needs a captured UI context must preserve or explicitly restore it; `ConfigureAwait(false)` is not a general instruction to run work on a background thread.
- Prefer `Task` over `ValueTask` unless an API contract or measured allocation need justifies the latter. Observe `ValueTask`'s consumption restrictions rather than assuming it can always be awaited repeatedly or shared like a `Task`.
- Define concurrency and ordering deliberately. Protect compound state transitions as a unit, bound parallel work and avoid holding a monitor lock across asynchronous operations. A concurrent collection does not make an entire multi-step workflow atomic.
- Make repeated input, stale results, timeouts and partial completion explicit where they affect behaviour. Avoid scheduling or timing tricks that merely make a race less likely.

References: [asynchronous scenarios][async], [cancellation][cancellation], [ConfigureAwait guidance][configure-await], [ValueTask][value-task], [thread-safe collections][concurrent-collections].

## Ownership, disposal and events

- Identify the owner and intended lifetime of disposable resources, subscriptions, timers, cancellation sources and background tasks. Finish or cancel activity safely before releasing resources it still uses.
- Use `using` or `await using` when the scope owns the resource and the target supports the relevant disposal contract. Garbage collection does not replace deterministic cleanup of owned resources.
- Respect DI lifetimes and disposal ownership. Do not retain a shorter-lived dependency in a longer-lived object without an explicit scope. Consumers normally do not dispose container-owned services; externally supplied instances need an explicit owner.
- Detach subscriptions to longer-lived publishers when the subscriber's lifetime ends, or use an appropriate weak subscription mechanism with clear semantics. Keep a handler reference when later unsubscription requires it.
- Prefer established resource wrappers such as `SafeHandle` for unmanaged handles. Do not add finalizers to types that only own managed resources. Implement disposal consistently with the type's inheritance and ownership contract.

References: [disposal][dispose], [dependency injection guidelines][di], [event subscriptions][events].

## Collections, LINQ and data handling

- Choose a collection for the required operations, ordering, uniqueness and scale. Do not repeatedly scan a list when keyed access is the actual contract, or add a cache without an owner and an invalidation rule.
- Understand when a LINQ query executes and whether it is enumerated more than once. Deferred queries can observe later state or re-run expensive work. Materialize deliberately when a snapshot or reuse is required, accounting for memory cost.
- Keep query predicates and projections free of surprising side effects. Use a clear loop when sequencing, mutation or early exit is the purpose. Choose between `First`, `Single` and their default-returning variants according to the required contract.
- Make string comparison explicit for identifiers and protocol values, normally with an appropriate ordinal comparison. Use culture-aware comparison and formatting where the meaning is linguistic or user-facing.
- Make units, numeric precision and time semantics explicit. Choose representations for instants, local schedules, durations and monetary quantities according to the domain; define required conversion and rounding behaviour at boundaries.
- Validate external data before it becomes trusted state. Enforce relevant ranges, sizes and required fields rather than assuming successful parsing or deserialization establishes business validity.

References: [deferred execution][deferred], [string comparisons][strings].

## Errors, logging and diagnostics

- Prefer the `Try` pattern for an expected unsuccessful lookup, parse or similar operation when a success flag and result fully describe the outcome. Use an explicit result when callers need additional outcome details, and exceptions for failures the contract represents as exceptional.
- Catch an exception when the boundary can recover, translate it meaningfully, preserve cleanup or report the failure. Avoid empty catches and returning apparent success after a failed or incomplete operation.
- Preserve diagnostic context. Use `throw;` to rethrow within a catch block, and retain the original exception when translating it into another exception type.
- Handle cancellation distinctly where it represents an expected outcome. Do not hide unrelated failures by treating every exception as cancellation or missing data.
- Use the established logging mechanism and consistent event context, including identifiers needed to connect related operations. Log failures at an appropriate owning boundary rather than repeating the same exception at every layer.
- Keep credentials, tokens and sensitive payloads out of logs. Capture the evidence needed to diagnose behaviour without indiscriminately dumping objects or external requests.

Reference: [exception handling][exceptions].

### The Try pattern

Use a descriptive `Try...` name, a Boolean success return and an `out` value for a synchronous operation whose expected unsuccessful outcome is part of normal use. Define precisely what success and failure mean.

For example, this contract distinguishes finding a setting from the setting's Boolean value:

```csharp
public interface IFeatureSettingReader
{
    bool TryGetFeatureEnabled(string featureName, out bool isEnabled);
}
```

- A return value of `true` means the setting was retrieved; `isEnabled` may legitimately be `false`. A return value of `false` means the documented lookup did not succeed. Use names that keep these meanings distinct.
- Specify the output on failure and assign it on every normal return path. Callers should normally consume the output only after success, unless the contract explicitly defines a useful failure value.
- Return `false` for the defined expected case. Do not catch every exception and return `false`, conceal invalid usage or lose unexpected I/O and programming failures.
- Keep ordinary Boolean questions, such as `IsEnabled`, when the value itself is the answer and there is no separate retrieval outcome.
- For asynchronous work, return a task containing a meaningful result. An `async` method cannot use `out` parameters; preserve the success/value distinction in its result instead.

Where the behaviour is at risk, test successful retrieval of both `true` and `false` values separately from unsuccessful retrieval.

References: [Try-Parse pattern][try-pattern], [method parameter restrictions][parameters].

## Readability, naming and language features

Names must communicate the object's responsibility, the value's meaning or the operation's purpose. Apply the following conventions alongside explicit project requirements, and encode the mechanical rules in `.editorconfig` or analyser configuration. Surface a conflict rather than silently substituting another style.

| Element | Default |
| --- | --- |
| Namespaces, types, methods, properties and events | Descriptive `PascalCase` names; interfaces use an `I` prefix. |
| Parameters and local variables | Descriptive `camelCase` names. |
| Constants | Descriptive `PascalCase` names. |
| Private instance fields | `_camelCase`; use `readonly` when the reference or value is assigned only during initialization. |
| Asynchronous methods | An `Async` suffix for task-based methods, except where an existing contract determines the name. |
| Layout | Four spaces, consistent braces and one statement per line; follow the repository's formatter. |

- Name classes for the responsibility they own, such as `InvoiceCalculator`, and methods for the action they perform, such as `CalculateTotal`. Name variables for the information they hold, such as `outstandingInvoices`. Avoid unexplained abbreviations, vague `Manager` or `Helper` names, and placeholders such as `data`, `obj` or `me` when a specific meaning is available.
- Give Boolean values readable meanings, using names such as `isEnabled`, `hasPendingChanges` or `canSave` when appropriate. Include units or relevant state in names where confusion would affect correctness.
- Use explicit variable types in production code, including built-in types and `out` declarations. Prefer target-typed construction such as `InvoiceCalculator invoiceCalculator = new();` where supported, retaining the type visibly on the declaration.
- Allow `var` selectively in unit tests when the type is apparent and it helps the scenario read clearly. A language-required case, such as a local holding an anonymous type, is also a valid exception. Do not replace explicit types with `var` across production code for brevity.
- Use pattern matching, expression-bodied members and newer syntax when they improve readability and match the project's toolchain. Prefer an ordinary block when compressed syntax obscures behaviour.
- Keep methods cohesive and control flow readable. Extract a method for a meaningful operation, not to meet an arbitrary line limit. Use guard clauses where they make valid and invalid paths clearer.
- Explain reasoning, contracts, constraints and non-obvious tradeoffs in comments. Document public APIs sufficiently for consumers, including ownership and exceptional behaviour. Avoid comments that merely repeat the next statement.
- Use supported source generators to remove repeated plumbing where worthwhile. Review their effects on the public contract and build, change the source declarations rather than generated output, and keep business behaviour visible.

References: [C# coding conventions][coding-style], [explicit declarations and target-typed new][declarations]. The explicit-type preference above is this repository's chosen convention.

## EditorConfig and automated enforcement

Use `.editorconfig` for project formatting and deliberate local conventions. The shared [`csharp.globalconfig`](csharp.globalconfig) holds the reusable analyser rules for explicit types, target-typed construction and naming. Reference it from the maintained checkout or a pinned submodule so there is one source for those rules.

Set `EngineeringStandardsRoot` to that checkout's actual path, then include this item in the consuming C# project or its `Directory.Build.props`:

```xml
<ItemGroup>
  <GlobalAnalyzerConfigFiles Include="$(EngineeringStandardsRoot)/csharp.globalconfig" />
</ItemGroup>
```

A cloned sibling repository is not automatically an EditorConfig parent. Global analyser configuration handles shared diagnostics; indentation and other editor formatting remain in the consuming project's `.editorconfig`.

To allow selective `var` use in unit tests, this narrow override can live in an `.editorconfig` inside the actual unit-test project directory:

```ini
[*.cs]
dotnet_diagnostic.IDE0008.severity = none
dotnet_diagnostic.IDE0007.severity = none
```

Keep the production rule active. Local `.editorconfig` values take precedence over the shared global configuration, so review overrides as explicit policy choices.

Use supported SDK/analyser tooling and enable `EnforceCodeStyleInBuild` when the project requires style diagnostics during builds. Set the relevant diagnostic severities and verify the intended rules run in both the editor and CI. Resolve new violations; do not suppress a rule merely to obtain a passing build. Stage adoption of the baseline in existing projects according to their actual toolchain and compatibility constraints.

Automate mechanical checks while retaining design review. Capitalization rules cannot determine whether a name describes its responsibility, and analyser success cannot establish SOLID, appropriate patterns or correct behaviour.

References: [analyser configuration files][analyser-config], [explicit-type rule][explicit-types], [target-typed construction rule][target-new], [naming rules][naming-rules], [build enforcement][style-build].

## Unit tests, integration tests and regression evidence

Select checks from requirements, contracts and supported failure scenarios. Design for separation so behaviour can be exercised without starting unrelated infrastructure. Keep effort proportionate to the risk; a coverage percentage or a passing build is not evidence for every behaviour.

### Unit tests

- Test public behaviour, business invariants, meaningful state transitions and relevant edge cases. A test should express what is required, not duplicate the implementation's calculations or private control flow.
- Exercise real objects for the behaviour under test. Control external services, time, randomness and events through suitable existing seams or simple fakes; avoid mocking every internal collaborator.
- Assert outcomes and contractually significant interactions. Avoid dependence on private methods, incidental call order or a particular arrangement of helper classes.
- Await asynchronous operations, control completion and avoid fixed sleeps. Cover cancellation, failure and stale-result handling where those are part of the contract.
- Keep ViewModel tests independent of views and real dispatcher execution. WPF-specific integration and automation limits are defined in [WPF.md](WPF.md#unit-tests-and-integration-tests).

### Integration tests

- Exercise the real collaboration across the boundary at risk, such as serialization, persistence, package integration or message handling. Preserve the relevant production components and substitute only the external systems that need control.
- Make setup, test data, isolation and cleanup reliable. Use existing test facilities; choose additional infrastructure for a demonstrated need and scope that work deliberately.
- Verify actual contracts and supported consumer behaviour, including relevant failures and compatibility. A test composed entirely of mocks cannot establish that the real integration works.

### Regression evidence and completion

- Reproduce a defect at the appropriate level and retain a focused regression check where practical. Confirm the check distinguishes the faulty behaviour from the correction.
- Build the affected targets and run the checks that can expose the relevant failure. Respect project gates and report unavailable environments or checks as unverified.
- Review responsibilities, invariants, dependency direction, lifetime and the purpose of each abstraction. A passing test suite does not replace this design review.
- Report the behaviour changed, significant decisions, checks actually performed and material uncertainty. Stop extending verification when the relevant risk is sufficiently resolved.

Reference: [unit testing guidance][unit-testing].

## Performance

- Measure the affected workload before choosing an optimisation. State the environment, inputs and comparison; distinguish latency, throughput, allocations and retained memory.
- Prefer an appropriate algorithm, data structure and ownership model before adding caching, pooling, unsafe code or low-level memory techniques.
- Use techniques such as spans, pooled buffers or specialised async return types when the benefit warrants their additional lifetime and correctness constraints. Keep a clear implementation when no meaningful benefit is demonstrated.
- Check that the change preserves required behaviour and improves the actual scenario. Avoid benchmarks that exclude the integration responsible for the original problem.

## References

These are supporting sources, not a requirement to read every page for every task. Select the source and API version relevant to the project. The standards above express this repository's engineering policy, including where it chooses a pragmatic default or requires a design judgement.

[language-version]: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/configure-language-version
[library-design]: https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/
[object-design]: https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/object-oriented/
[nullable]: https://learn.microsoft.com/en-us/dotnet/csharp/nullable-references
[records]: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/record
[readonly-collection]: https://learn.microsoft.com/en-us/dotnet/api/system.collections.objectmodel.readonlycollection-1
[di]: https://learn.microsoft.com/en-us/dotnet/core/extensions/dependency-injection/guidelines
[async]: https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/async-scenarios
[cancellation]: https://learn.microsoft.com/en-us/dotnet/standard/threading/cancellation-in-managed-threads
[configure-await]: https://devblogs.microsoft.com/dotnet/configureawait-faq/
[value-task]: https://learn.microsoft.com/en-us/dotnet/api/system.threading.tasks.valuetask
[concurrent-collections]: https://learn.microsoft.com/en-us/dotnet/standard/collections/thread-safe/
[dispose]: https://learn.microsoft.com/en-us/dotnet/standard/garbage-collection/implementing-dispose
[events]: https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/events/how-to-subscribe-to-and-unsubscribe-from-events
[deferred]: https://learn.microsoft.com/en-us/dotnet/standard/linq/deferred-execution-lazy-evaluation
[strings]: https://learn.microsoft.com/en-us/dotnet/standard/base-types/best-practices-strings
[exceptions]: https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions
[coding-style]: https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/coding-style/coding-conventions
[unit-testing]: https://learn.microsoft.com/en-us/dotnet/core/testing/unit-testing-best-practices
[try-pattern]: https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/exceptions-and-performance
[parameters]: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/method-parameters
[declarations]: https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/declarations
[analyser-config]: https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/configuration-files
[explicit-types]: https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/ide0007-ide0008
[target-new]: https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/ide0090
[naming-rules]: https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/naming-rules
[style-build]: https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/overview#code-style-analysis
