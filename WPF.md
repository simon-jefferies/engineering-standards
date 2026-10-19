# WPF engineering standards

Use this guide when designing, implementing or reviewing Windows Presentation Foundation applications and libraries. Apply the sections relevant to the change. These are engineering standards for people and coding agents; the Microsoft references explain the platform behaviour behind them.

Last reviewed: 19 October 2026.

## Core expectation

Deliver a working implementation with coherent responsibilities, encapsulated state, explicit dependencies and deliberate resource ownership. Treat these qualities as completion criteria throughout the work. Explain material departures and their consequences.

Choose the smallest coherent design that meets the requirement. Class counts, interface counts and line counts do not establish design quality.

## Before changing code

- Establish the expected behaviour, the affected feature and the evidence for the change. Separate observed facts from assumptions.
- Check the target framework, language version, supported Windows environments and existing MVVM toolkit. Keep changes compatible with the project's toolchain, including .NET Framework where applicable.
- Identify the current ownership and intended architecture. Existing code is evidence of implementation, not proof of the desired design. Keep improvements within the requested scope and explain conflicts with explicit project constraints.
- Use established binding, command, dependency injection and logging mechanisms where suitable. Explain the concrete need before adding another framework, event bus, project or dependency.

## Object design and MVVM responsibilities

Use these responsibilities as a design guide, not a requirement to create a separate class or project for every row.

| Area | Responsibility |
| --- | --- |
| View and view-specific code | Layout, styling, focus, visual interaction and control-specific behaviour. |
| ViewModel | Presentation state, selection, validation feedback, commands and coordination of application operations. |
| Domain or application objects | Business rules, valid state transitions and operations that have meaning independently of a particular view. |
| Infrastructure adapters | Storage, networking, device access and other external integrations. |
| Composition and lifetime owner | Constructing collaborators and deciding their scopes, activation and cleanup. |

- Put behaviour with the object responsible for it. Avoid adding unrelated work to the most convenient ViewModel or collecting it in an unstructured helper class.
- Protect business invariants through meaningful operations. Editable bindable properties and plain data transfer objects are legitimate; callers must not bypass domain rules by coordinating arbitrary setters or mutating exposed collections.
- Keep dependencies explicit. Prefer supplying collaborators over hidden service lookups and global mutable state. Introduce interfaces for meaningful contracts, substitution or external boundaries, rather than for every class.
- Use composition for independent behaviours. Use inheritance where the derived type honours the base contract; a focused ViewModel base class is reasonable, a hierarchy of unrelated features is not.
- Keep domain and reusable application logic independent of WPF types. Keep direct access to controls, windows and the visual tree in the presentation integration layer.
- Allow focused code-behind for view concerns such as focus or animation. Moving that code into a behaviour or service must have a concrete benefit. Business decisions, persistence and application workflows need an appropriate owner outside the view.

## State, bindings and commands

- Use the project's notification mechanism for changing bound properties. Notify dependent calculated properties when their inputs change. Keep property getters cheap and free of side effects.
- Use collection change notifications when membership changes must appear in the UI. `ObservableCollection<T>` notifications do not replace property notifications on the individual items.
- Understand the binding source, `DataContext`, mode and update trigger. Specify mode and trigger where the choice affects editing, validation or correctness; do not assume every control updates the source immediately.
- Preserve a reusable control's consumer-facing `DataContext`. Bind its internal elements to its own properties through an explicit source, such as `ElementName` or `RelativeSource`, when appropriate.
- Keep converters focused on representation. Business rules and external operations need explicit owners rather than being hidden in converters or property setters.
- Use commands for application actions. Refresh availability through the command implementation's notification mechanism when relevant state changes. Keep UI availability and the operation's own validity checks consistent.
- Verify the actual disabled state and the corresponding appearance in the control template. Changing colour or opacity alone does not prevent interaction.
- Trace binding failures to their source. Avoid masking missing state with arbitrary fallback values, repeated refreshes or direct writes to bound controls.

References: [data binding][binding], [commands][commands].

## Threading and asynchronous work

- Access thread-affine WPF objects through their owning UI thread. Keep dispatched updates small; do not assume every window uses one global dispatcher.
- Await asynchronous work instead of blocking the UI thread with `.Wait()` or `.Result()`. Use asynchronous APIs directly for I/O; use `Task.Run` for appropriate CPU work or identified blocking work that can safely run elsewhere. Adding `async` alone does not move work to another thread.
- Return awaitable operations from application code. Reserve `async void` for framework-required event handlers, with error handling at that boundary. Keep constructors free of untracked asynchronous work.
- Define busy state, repeated-command behaviour, cancellation and failure reporting. Prevent late or superseded results from updating a replaced or inactive presentation. Give background tasks an owner that observes their completion and failures.
- Default to updating UI-bound collections on the owning UI thread. `ObservableCollection<T>` is not a cross-thread synchronization policy. Where background mutation is justified, coordinate application and WPF access using an explicit strategy such as `BindingOperations.EnableCollectionSynchronization`; follow its registration, locking and notification requirements.
- Put thread switching at the relevant presentation boundary. Keep reusable application logic independent of a particular dispatcher.

References: [WPF threading][threading], [asynchronous scenarios][async], [collection synchronization][collection-sync].

## Ownership and lifetime

- Identify the owner and intended lifetime of subscriptions, timers, cancellation sources, background tasks and disposable resources. End their activity and release owned resources when that lifetime ends.
- For subscriptions to longer-lived publishers, provide a detachment path or justify a weak event mechanism. Weak references do not cancel work, dispose resources or define application lifetime.
- Treat `Loaded` and `Unloaded` as attachment changes. A view can unload and load again; `Unloaded` is not a universal shutdown callback. Make activation repeatable without duplicate subscriptions. Dispose a ViewModel according to its owner's lifetime contract, not merely because one associated view unloaded.
- Dispose only resources the object owns. Account for shared instances, cached views, navigation, media and native resources where relevant.

References: [weak events][weak-events], [Unloaded semantics][unloaded].

## XAML and reusable controls

- Use styles, resources and templates to express visual consistency. Prefer a `UserControl` for a composed view; choose a templatable custom control when reusable behaviour needs independently replaceable visuals.
- Expose a meaningful control contract through properties, commands and events. Avoid making consumers depend on internal element names or a concrete application ViewModel.
- Use dependency properties for control properties that need WPF property-system features. Keep CLR wrappers thin and use the appropriate metadata callbacks, validation or coercion for property-system behaviour.
- Keep templates and resource dictionaries focused. Verify resource lookup, template bindings and interactive states when changing them.
- Respect the project's resizing, DPI, localisation and theme requirements. Preserve keyboard access, focus indication and useful accessibility information in custom controls.

References: [control authoring][controls], [dependency properties][properties].

## Performance and evidence

- Measure the affected scenario before selecting a performance fix. Distinguish startup cost, interaction latency, rendering and retained memory; report the environment and workload with comparisons.
- Preserve virtualization for sizeable item displays. Check panels, scrolling and templates before assuming it is active. Account for recycled containers so one item's state is not displayed for another. UI virtualization does not remove the underlying dataset from memory.
- Keep frequent binding evaluation and UI callbacks inexpensive. Batch or throttle high-volume updates when appropriate, with explicit behaviour for delayed or discarded updates.
- Investigate retention and resource ownership before introducing caches, pooling or forced garbage collection as a response to memory growth.

Reference: [control performance][performance].

## Verification and completion

Select checks that can expose a realistic failure in the changed behaviour. Keep verification proportionate to the risk and use the project's established test facilities.

- Verify business rules and presentation state transitions independently where useful. Add or adapt focused automated checks when they protect the behaviour at risk.
- For a WPF integration change, exercise the actual binding, command, template, layout or lifecycle involved. A passing ViewModel test cannot establish that the rendered control behaves correctly.
- Where affected, check repeated activation, cancellation, rapid user input, stale results, empty data, failures, disposal and the required display configurations. Record manual verification when automation is unavailable.
- Review the resulting design: responsibilities, invariants, dependency direction, abstraction purpose and lifetime ownership. Address material weaknesses found within the agreed scope.
- Report the behaviour changed, significant design decisions, checks actually performed and remaining uncertainty. State clearly when Windows execution or a required environment was unavailable. Do not report unperformed checks as passing.

### Example: an action should become disabled

Identify the object that owns the eligibility rule. Expose the corresponding presentation state or command availability, notify when it changes, and let the view reflect it. Verify that the operation rejects invalid execution and that the actual control becomes disabled and looks disabled. If these disagree, trace the rule, notification, binding and template before adding a visual workaround.

## References

The links below are supporting documentation, not instructions to read every page before every change. Select the reference relevant to the work and the API version supported by the project.

[binding]: https://learn.microsoft.com/en-us/dotnet/desktop/wpf/data/
[commands]: https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/commanding-overview
[threading]: https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/threading-model
[async]: https://learn.microsoft.com/en-us/dotnet/csharp/asynchronous-programming/async-scenarios
[collection-sync]: https://learn.microsoft.com/en-us/dotnet/api/system.windows.data.bindingoperations.enablecollectionsynchronization
[weak-events]: https://learn.microsoft.com/en-us/dotnet/desktop/wpf/events/weak-event-patterns
[unloaded]: https://learn.microsoft.com/en-us/dotnet/api/system.windows.frameworkelement.unloaded
[controls]: https://learn.microsoft.com/en-us/dotnet/desktop/wpf/controls/control-authoring-overview
[properties]: https://learn.microsoft.com/en-us/dotnet/desktop/wpf/properties/custom-dependency-properties
[performance]: https://learn.microsoft.com/en-us/dotnet/desktop/wpf/advanced/optimizing-performance-controls
