# amplifier-bundle-superpowers

A development methodology bundle for [Amplifier](https://github.com/microsoft/amplifier) based on the [Superpowers](https://github.com/obra/superpowers) framework by Jesse Vincent. Provides TDD enforcement, subagent-driven development, and design-before-code workflows.

## Overview

This bundle brings the Superpowers development methodology to Amplifier:

- **Design Before Code** - Brainstorm and validate designs before touching code
- **Test-Driven Development** - RED-GREEN-REFACTOR with strict enforcement
- **Subagent-Driven Development** - Fresh agent per task with two-stage review
- **Verification Before Completion** - Prove it works, don't just claim it

## Quick Start

```bash
# Add the bundle
amplifier bundle add git+https://github.com/microsoft/amplifier-bundle-superpowers@main --name superpowers

# Activate it
amplifier bundle use superpowers

# Start a session
amplifier
```

## The Workflow

```
/brainstorm → Design Document
     ↓
/write-plan → Implementation Plan (bite-sized tasks)
     ↓
/execute-plan → Subagent-Driven Development
     ↓
     Per Task:
     1. Implementer agent (implements + tests + commits)
     2. Spec reviewer agent (validates against spec)
     3. Code quality reviewer agent (ensures quality)
     ↓
Finished → PR or Merge
```

## Modes

The bundle provides three workflow modes:

| Mode | Shortcut | Purpose |
|------|----------|---------|
| `brainstorm` | `/brainstorm` | Design refinement - explore approaches and trade-offs |
| `write-plan` | `/write-plan` | Create detailed implementation plan with TDD tasks |
| `execute-plan` | `/execute-plan` | Execute plan with subagent-driven development |

Use `/modes` to list all available modes, `/mode off` to exit a mode.

## Agents

| Agent | Purpose |
|-------|---------|
| `superpowers:brainstormer` | Facilitates design refinement through dialogue |
| `superpowers:plan-writer` | Creates detailed implementation plans |
| `superpowers:implementer` | Implements tasks following strict TDD |
| `superpowers:spec-reviewer` | Reviews code against spec compliance |
| `superpowers:code-quality-reviewer` | Reviews code quality and best practices |

## Skills Library

The original [Superpowers skills](https://github.com/obra/superpowers) are included and available via the skills tool:

- **test-driven-development** - RED-GREEN-REFACTOR cycle
- **systematic-debugging** - 4-phase root cause analysis
- **brainstorming** - Design refinement process
- **writing-plans** - Implementation plan creation
- **subagent-driven-development** - Task execution with reviews
- **verification-before-completion** - Prove it works

Use `load_skill(search="superpowers")` to discover all available skills.

## Composing Just the Methodology

If you want to add the Superpowers agents and methodology to your own bundle without replacing your entire configuration, include just the behavior:

```yaml
includes:
  - bundle: git+https://github.com/microsoft/amplifier-bundle-superpowers@main#subdirectory=behaviors/superpowers-methodology.yaml
```

This gives you the 5 agents and methodology context without changing your providers, tools, or other configuration.

## Bundle Structure

```
amplifier-bundle-superpowers/
├── bundle.md                              # Root bundle (thin pattern)
├── behaviors/
│   └── superpowers-methodology.yaml       # Composable behavior
├── agents/
│   ├── brainstormer.md                    # Design refinement
│   ├── plan-writer.md                     # Implementation planning
│   ├── implementer.md                     # TDD implementation
│   ├── spec-reviewer.md                   # Spec compliance review
│   └── code-quality-reviewer.md           # Code quality review
├── modes/
│   ├── brainstorm.md                      # /brainstorm mode
│   ├── write-plan.md                      # /write-plan mode
│   └── execute-plan.md                    # /execute-plan mode
├── context/
│   └── superpowers-methodology.md         # Core methodology
└── recipes/
    └── subagent-driven-development.yaml   # Workflow recipe
```

## Acknowledgments

This bundle is an Amplifier adaptation of [Superpowers](https://github.com/obra/superpowers) by [Jesse Vincent](https://github.com/obra), licensed under MIT. The original project provides a skills framework and software development methodology for coding agents.

## Contributing

This project welcomes contributions and suggestions. Most contributions require you to agree to a
Contributor License Agreement (CLA) declaring that you have the right to, and actually do, grant us
the rights to use your contribution. For details, visit https://cla.opensource.microsoft.com.

When you submit a pull request, a CLA bot will automatically determine whether you need to provide
a CLA and decorate the PR appropriately (e.g., status check, comment). Simply follow the instructions
provided by the bot. You will only need to do this once across all repos using our CLA.

This project has adopted the [Microsoft Open Source Code of Conduct](https://opensource.microsoft.com/codeofconduct/).
For more information see the [Code of Conduct FAQ](https://opensource.microsoft.com/codeofconduct/faq/) or
contact [opencode@microsoft.com](mailto:opencode@microsoft.com) with any additional questions or comments.

---

## Trademarks

This project may contain trademarks or logos for projects, products, or services. Authorized use of Microsoft
trademarks or logos is subject to and must follow
[Microsoft's Trademark & Brand Guidelines](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general).
Use of Microsoft trademarks or logos in modified versions of this project must not cause confusion or imply Microsoft sponsorship.
Any use of third-party trademarks or logos are subject to those third-party's policies.
