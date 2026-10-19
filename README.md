# Engineering standards

Practical standards for designing, changing and reviewing software, written for developers and coding agents.

A working feature should also have coherent responsibilities, encapsulated state, understandable dependencies and deliberate resource ownership. These guides make those expectations explicit without prescribing unnecessary layers or abstractions.

## Guides

| Guide | Use it for |
| --- | --- |
| [WPF](WPF.md) | Windows Presentation Foundation applications and libraries: OO design, MVVM, bindings, commands, threading, lifetime, controls and verification. |

## Using the guidance

Keep the guides in this repository as the maintained source. Reference the relevant guide from your coding tool's project or personal instructions, using a shared checkout, an initialized submodule or a supported retrieval mechanism.

For example, an instruction for a coding agent could say:

> When designing, implementing or reviewing WPF code, read WPF.md from the engineering-standards checkout and apply the relevant sections alongside this project's requirements. State explicitly if the guide could not be loaded.

Include the actual path or URL in your tool's configuration. A Markdown link alone does not guarantee that a tool loads its target. Confirm the guide was read in the environment where the agent runs.

Project-specific behaviour, supported versions, build commands and explicit exceptions belong with the consuming project. Where a guide conflicts with an explicit project requirement, explain the conflict and the chosen resolution.

Use a tag or commit when a project needs a fixed revision. Updating this repository does not refresh existing clones or already-running agent sessions automatically.

## Improving the standards

Propose changes based on concrete implementation or review experience. Explain what a rule protects, when it applies and any reasonable exceptions. Keep framework facts supported by primary documentation, and keep examples free of private project or customer details.
