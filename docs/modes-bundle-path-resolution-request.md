# Feature Request: Bundle-Aware Mode Discovery for Custom Bundles

## Summary

When a third-party bundle declares custom modes and references `hooks-mode` via git source, the custom modes are not discovered. Only the built-in modes (`explore`, `plan`, `careful`) from the modes bundle itself appear.

## Context

We built an Amplifier bundle ([bkrabach/superpowers](https://github.com/bkrabach/superpowers)) that provides a development methodology (TDD, subagent-driven development, design-before-code). As part of this bundle, we created three custom modes:

- `/brainstorm` - Design refinement before implementation
- `/write-plan` - Create detailed implementation plans
- `/execute-plan` - Subagent-driven development with two-stage review

These mode files live at `modes/brainstorm.md`, `modes/write-plan.md`, and `modes/execute-plan.md` within our bundle repository and follow the correct mode file format with `mode:` frontmatter, shortcuts, tool policies, and context.

## Bundle Configuration

Our `bundle.md` references hooks-mode from the modes bundle:

```yaml
hooks:
  - module: hooks-mode
    source: git+https://github.com/microsoft/amplifier-bundle-modes@main#subdirectory=modules/hooks-mode
    config:
      search_paths:
        - modes
```

## What Happens

When a user installs and activates our bundle:

```
> /modes
Available modes:
  careful — Full capability with confirmation for destructive actions
  explore — Zero-footprint exploration - understand before acting
  plan — Analyze, strategize, and organize - but don't implement

> /brainstorm
Unknown command: /brainstorm. Use /help for available commands.
```

The three built-in modes are found. Our custom modes are not.

## Root Cause Analysis

We traced the issue to how `hooks-mode` discovers mode files. There are three discovery mechanisms:

1. **Project-local**: `{working_dir}/.amplifier/modes/` - auto-discovered
2. **User-global**: `~/.amplifier/modes/` - auto-discovered
3. **Bundle built-in**: `__file__` walk up 4 levels from module code to find sibling `modes/` directory

The third mechanism is what finds the built-in modes. However, when a third-party bundle references hooks-mode via git source, the module code lives in the **cached modes bundle** directory, not the third-party bundle's directory. The `__file__` walk finds the modes bundle root (with `explore.md`, `plan.md`, `careful.md`) rather than our bundle root (with `brainstorm.md`, `write-plan.md`, `execute-plan.md`).

The `search_paths` config option treats entries as raw filesystem paths (`Path(path_str)`) with no bundle-aware resolution. A relative path like `modes` resolves against `cwd`, not the bundle's installed location.

## Current Workarounds

Users can manually copy mode files to `~/.amplifier/modes/` or `{project}/.amplifier/modes/`, but this breaks the "install a bundle and everything works" experience.

## What Would Help

Some mechanism for bundles that include `hooks-mode` to also contribute mode definitions that get discovered automatically. We don't have a specific solution in mind - we'd defer to your judgment on what fits the modes bundle's design philosophy. A few directions we considered but didn't pursue:

- Bundle-aware path resolution in `search_paths` config
- A way for bundles to register additional mode directories during mount
- Some form of mode composition through the bundle system
- A convention that hooks-mode checks for modes in all bundles that include it

## Environment

- Amplifier CLI (latest)
- amplifier-bundle-modes@main
- Tested on macOS (Apple Silicon) and Linux (aarch64)
- Bundle source: https://github.com/bkrabach/superpowers

## Reproduction

```bash
# Install the superpowers bundle
amplifier bundle add git+https://github.com/bkrabach/superpowers@main --name superpowers
amplifier bundle use superpowers

# Start a session
amplifier

# Check modes - brainstorm/write-plan/execute-plan are missing
/modes
```

The mode files exist in the bundle repo at `modes/*.md` and are correctly formatted - they just aren't discovered.
