# Engineering standards

Practical standards for designing, changing and reviewing software, written for developers and coding agents.

A working feature should also have coherent responsibilities, encapsulated state, understandable dependencies and deliberate resource ownership. These guides make those expectations explicit without prescribing unnecessary layers or abstractions.

## Guides

| Guide | Use it for |
| --- | --- |
| [C#](CSHARP.md) | Shared OOP and SOLID principles, composition, naming, explicit types, contracts, Try methods, dependencies, async work, ownership and tests. |
| [WPF](WPF.md) | WPF-specific guidance alongside the C# standards: ViewModel separation, CommunityToolkit.Mvvm, bindings, dispatchers, controls and integration verification. |

## Using the guidance

Keep the guides in this repository as the maintained source. Reference the relevant guide from your coding tool's project or personal instructions, using a shared checkout, an initialized submodule or a supported retrieval mechanism.

Read `CSHARP.md` for C# work. For WPF work, read `WPF.md` as well; its common object-design rules refer back to the C# guide.

For example, an instruction for a coding agent could say:

> When designing, implementing or reviewing C# code, read CSHARP.md from the engineering-standards checkout. For WPF work, also read WPF.md. Apply the relevant sections alongside this project's requirements. State explicitly if a required guide could not be loaded.

Include the actual path or URL in your tool's configuration. A Markdown link alone does not guarantee that a tool loads its target. Confirm the guide was read in the environment where the agent runs.

Project-specific behaviour, supported versions, build commands and explicit exceptions belong with the consuming project. Where a guide conflicts with an explicit project requirement, explain the conflict and the chosen resolution.

Use a tag or commit when a project needs a fixed revision. Updating this repository does not refresh existing clones or already-running agent sessions automatically.

## Shared C# analyser configuration

[csharp.globalconfig](csharp.globalconfig) supplies mechanical naming and explicit-type rules, including a preference for target-typed `new` where supported. Reference it from a consuming C# project through `GlobalAnalyzerConfigFiles`; the [C# guide explains setup and unit-test exceptions](CSHARP.md#editorconfig-and-automated-enforcement).

Keep formatting and intentional local overrides in the consuming project's `.editorconfig`. Merely cloning this repository does not apply these rules elsewhere. Semantic naming, OOP, SOLID and appropriate pattern use remain required design-review responsibilities.

## Improving the standards

Propose changes based on concrete implementation or review experience. Explain what a rule protects, when it applies and any reasonable exceptions. Keep framework facts supported by primary documentation, and keep examples free of private project or customer details.
