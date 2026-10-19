# WPF engineering standards

Use this guide when designing, implementing or reviewing Windows Presentation Foundation applications and libraries. Apply the sections relevant to the change. These are engineering standards for people and coding agents; the Microsoft references explain the platform behaviour behind them.

Last reviewed: 19 October 2026.

## Core expectation

Deliver a working implementation with coherent responsibilities, encapsulated state, explicit dependencies and deliberate resource ownership. Treat these qualities as completion criteria throughout the work. Explain material departures and their consequences.

Keep ViewModels separate from views. Their construction, state changes and commands must be testable without loading XAML, creating controls, starting a WPF `Application` or running a real `Dispatcher`.

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
- ViewModels expose presentation state and actions without creating or retaining view instances. Put view-specific operations, such as showing a window or moving focus, in view integration. Supply a focused contract when the ViewModel needs to request such an operation; do not discover views or services through `Application.Current`.
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

## CommunityToolkit.Mvvm

Consider `CommunityToolkit.Mvvm` for new WPF work needing MVVM support. In existing applications, use the chosen toolkit consistently and justify a migration or another overlapping framework. The toolkit supplies notification and command infrastructure; responsibility boundaries still need deliberate design.

- Use `ObservableObject` and `SetProperty` for observable state where appropriate. Reuse the project's existing compatible base class rather than introducing another hierarchy solely for notification plumbing.
- Prefer `[ObservableProperty]` and `[RelayCommand]` where source generation is supported and fits the project. Keep the containing types `partial` as required, and check the installed package, compiler and generator support before choosing syntax from newer examples. Explicit properties and command instances remain valid when needed.
- Change generated properties through their property setters when notifications are required; writing their backing fields bypasses the generated notification logic.
- Use `[NotifyPropertyChangedFor]` for dependent properties and `[NotifyCanExecuteChangedFor]` or `NotifyCanExecuteChanged()` for affected command availability. The toolkit does not automatically discover changes to a `CanExecute` condition.
- Use `AsyncRelayCommand` or `[RelayCommand]` on a `Task`-returning method for asynchronous actions. Define cancellation, concurrent execution and failure handling deliberately. Use exposed state such as `IsRunning` when it represents the required busy state, and pass cancellation tokens through to the work that supports cancellation.
- Handle expected failures at the operation boundary and expose useful presentation feedback. Do not assume an async command absorbs exceptions, or enable `FlowExceptionsToTaskScheduler` without an owner that observes failures.
- Keep messenger use purposeful. Prefer explicit collaborators for a direct request and response. If using `IMessenger`, make its scope and recipient lifetime explicit and avoid introducing another global event bus merely because the toolkit provides one.
- Test the ViewModel through its public properties and commands. Await `IAsyncRelayCommand.ExecuteAsync` in asynchronous tests; assert the application's state and command behaviour without constructing a view or testing the toolkit's generated implementation itself.

References: [MVVM Toolkit overview][mvvm], [ObservableObject][observable-object], [observable property generation][observable-property], [command generation][relay-command], [async commands][async-command], [messenger][messenger].

## Threading and asynchronous work

- Await asynchronous work instead of blocking the UI thread with `.Wait()` or `.Result()`. Use asynchronous APIs directly for I/O; use `Task.Run` for appropriate CPU work or identified blocking work that can safely run elsewhere. Adding `async` alone does not move work to another thread.
- Return awaitable operations from application code. Reserve `async void` for framework-required event handlers, with error handling at that boundary. Keep constructors free of untracked asynchronous work.
- Define busy state, repeated-command behaviour, cancellation and failure reporting. Prevent late or superseded results from updating a replaced or inactive presentation. Give background tasks an owner that observes their completion and failures.
- Default to updating UI-bound collections on the owning UI thread. `ObservableCollection<T>` is not a cross-thread synchronization policy. Where background mutation is justified, coordinate application and WPF access using an explicit strategy such as `BindingOperations.EnableCollectionSynchronization`; follow its registration, locking and notification requirements.

References: [WPF threading][threading], [asynchronous scenarios][async], [collection synchronization][collection-sync].

### Dispatcher usage

- Keep real WPF dispatcher access in view integration or a focused presentation adapter. Arrange background callbacks to enter presentation state through the established scheduling boundary. ViewModel business and presentation decisions must remain testable without a real dispatcher; do not add a dispatcher wrapper to every class.
- Use the dispatcher that owns the affected WPF object. Obtain it from that object or the established presentation owner. `Dispatcher.CurrentDispatcher` on a worker thread returns that thread's dispatcher, creating one if needed; it does not locate the UI thread. Do not assume `Application.Current.Dispatcher` owns every window.
- Use `CheckAccess()` when a boundary may need to marshal an incoming call. Use `VerifyAccess()` to enforce a documented UI-thread requirement. Neither method moves execution to another thread.
- Prefer `InvokeAsync` for short queued presentation updates, and await completion when subsequent work or error handling depends on them. Its callback still runs on the UI thread. Keep expensive work outside the callback and justify synchronous `Invoke` where required.
- Keep dispatched callbacks simple. If dispatching a `Func<Task>`, observe the inner task as well as the dispatch operation so completion and failures are not lost. Avoid passing an async lambda as an `Action`, which would create `async void` work.
- Use dispatcher priority for a defined scheduling need. Repeated dispatching, priority changes, sleeps or nested message pumps must not be used to conceal binding failures or race conditions. Account for queued work when its view is deactivated or its dispatcher shuts down.

References: [WPF threading][threading], [CurrentDispatcher][current-dispatcher], [thread access checks][dispatcher-access], [InvokeAsync][dispatcher-invoke].

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

### Dependency property generation

Consider the WPF-compatible [`DependencyPropertyGenerator` NuGet package][dp-generator-package] when repeated dependency-property declarations would benefit from source generation. This is an optional implementation choice for controls and attached properties. Ordinary ViewModel state should remain ordinary observable properties.

- Check the selected package's compiler, source-generator and target-framework compatibility, including CI builds. Use syntax supported by the project; do not enable preview language features merely to copy a newer example.
- Review the generated registration, owner type, CLR wrapper, defaults, binding metadata and change callbacks. Preserve validation, coercion and read-only semantics where required, using explicit declarations when the generator cannot express the needed contract. Change the source declarations and callbacks, not generated output.
- Keep callbacks focused on control behaviour and use the same ownership and thread-access rules as handwritten dependency properties. Avoid shared mutable metadata defaults; consider instance ownership when a property holds a collection or another mutable object.
- Adopt generation where it improves clarity and maintenance. It does not justify converting ViewModels to `DependencyObject`, adding a WPF runtime requirement to their unit tests or broadly rewriting existing controls.

References: [package documentation][dp-generator-source], [WPF dependency properties][properties].

## Performance and evidence

- Measure the affected scenario before selecting a performance fix. Distinguish startup cost, interaction latency, rendering and retained memory; report the environment and workload with comparisons.
- Preserve virtualization for sizeable item displays. Check panels, scrolling and templates before assuming it is active. Account for recycled containers so one item's state is not displayed for another. UI virtualization does not remove the underlying dataset from memory.
- Keep frequent binding evaluation and UI callbacks inexpensive. Batch or throttle high-volume updates when appropriate, with explicit behaviour for delayed or discarded updates.
- Investigate retention and resource ownership before introducing caches, pooling or forced garbage collection as a response to memory growth.

Reference: [control performance][performance].

## Unit tests and integration tests

Separation is the governing principle: test ViewModel behaviour without a view, and verify WPF wiring at the appropriate integration boundary. Choose checks from requirements, contracts and supported regression scenarios. Keep effort proportionate to the risk; test counts and coverage percentages do not establish correctness.

### Unit tests: ViewModels without views

- Construct the ViewModel directly with suitable collaborators. Exercise its public actions and commands, then assert observable state, results, validation and command availability. Test domain rules and application operations independently where they own the behaviour.
- Keep views, XAML loading, real dispatcher execution and WPF `Application` startup out of the unit-test suite. Do not add STA attributes, custom dispatcher pumps or view instances to make a ViewModel unit test run. If it requires these, inspect the dependency boundary and separate the behaviour under test from WPF integration.
- Where relevant, cover dependent property notifications, `CanExecuteChanged`, busy state, cancellation, failure, repeated input and superseded results. Test notifications that consumers rely on, rather than retesting every generated setter.
- Control application services, time and external events through existing meaningful contracts or simple fakes. Assert outcomes and contractually significant interactions; avoid coupling tests to private methods, incidental call order or the exact arrangement of helper objects.
- Await asynchronous operations and control their completion explicitly. Do not use fixed sleeps to guess when work has finished. A substitute scheduling dependency may support isolated ViewModel tests, but it does not establish that WPF dispatching works.

### Integration tests

- Exercise the real cooperating components across the boundary at risk, such as a ViewModel with its application service or an adapter with a controlled external system. Integration tests that do not involve WPF need no view or dispatcher.
- Keep view and real dispatcher tests in a separate WPF integration suite, using the project's established Windows/WPF host where available. Such a host needs the appropriate STA thread and a functioning dispatcher/message loop for queued UI work. Marking an ordinary test STA alone does not provide that environment.
- For a WPF change, exercise the actual XAML, `DataContext`, bindings, commands, resources, template or lifecycle implicated by it. Await meaningful completion with bounded timeouts. Verify the actual control state and the required visual behaviour; a ViewModel assertion cannot establish those results.
- Keep the integration host responsible for view cleanup, owned resources, asynchronous failures and dispatcher lifetime. Do not make ordinary unit tests depend on shared windows or application state.

### When view automation is impractical

Use established integration facilities where they can provide reliable evidence. Do not create a bespoke UI testing framework, add test-only behaviour to production code or add fragile STA and timing workarounds as part of routine feature or bug work. Scope any substantial test-infrastructure work as a deliberate engineering task.

When suitable automation or the required Windows environment is unavailable, record a focused manual verification scenario and its result, or state that it remains unverified. Include the relevant theme, display configuration, interaction and lifecycle conditions. Limited view automation is a reason to preserve separation and report the remaining gap honestly.

References for WPF hosting requirements: [WPF threading][threading], [dispatcher message loop][dispatcher-run].

## Verification and completion

- For a bug fix, reproduce the failure at the level where it occurs and retain a useful regression check where practical. Confirm that the check distinguishes the faulty behaviour from the correction.
- Select and run the relevant unit tests, integration tests and manual checks described above. Where affected, cover repeated activation, cancellation, rapid user input, stale results, empty data, failures, disposal and the required display configurations.
- Review the resulting design: responsibilities, invariants, dependency direction, abstraction purpose and lifetime ownership. Address material weaknesses found within the agreed scope.
- Report the behaviour changed, significant design decisions, checks actually performed and remaining uncertainty. State clearly when Windows execution or a required environment was unavailable. Do not report unperformed checks as passing.

### Example: an action should become disabled

Identify the object that owns the eligibility rule. Expose the corresponding presentation state or command availability, notify when it changes, and let the view reflect it. Unit-test the rule, the ViewModel's command availability and the relevant notification without creating a button. Separately, use WPF integration testing or recorded manual verification to check that the actual control becomes disabled and looks disabled. If these disagree, trace the rule, notification, binding and template before adding a visual workaround.

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
[mvvm]: https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/
[observable-object]: https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/observableobject
[observable-property]: https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/generators/observableproperty
[relay-command]: https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/generators/relaycommand
[async-command]: https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/asyncrelaycommand
[messenger]: https://learn.microsoft.com/en-us/dotnet/communitytoolkit/mvvm/messenger
[current-dispatcher]: https://learn.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatcher.currentdispatcher
[dispatcher-access]: https://learn.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatcher.checkaccess
[dispatcher-invoke]: https://learn.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatcher.invokeasync
[dispatcher-run]: https://learn.microsoft.com/en-us/dotnet/api/system.windows.threading.dispatcher.run
[dp-generator-package]: https://www.nuget.org/packages/DependencyPropertyGenerator
[dp-generator-source]: https://github.com/HavenDV/DependencyPropertyGenerator
