# Superpowers → Amplifier Bundle Conversion

## Current Focus
**COMPLETE** - Bundle created and ready for testing.

## Final Bundle Structure

```
superpowers/
├── bundle.md                    # Root bundle (thin, inherits foundation)
├── context/
│   └── superpowers-methodology.md   # Core TDD/verification principles
├── agents/
│   ├── implementer.md           # Task implementation (TDD)
│   ├── spec-reviewer.md         # Spec compliance review
│   ├── code-quality-reviewer.md # Quality review
│   ├── brainstormer.md          # Design refinement
│   └── plan-writer.md           # Implementation plans
├── recipes/
│   └── subagent-driven-development.yaml
├── commands/
│   ├── brainstorm.md
│   ├── write-plan.md
│   └── execute-plan.md
└── superpowers-upstream/        # Git submodule with original skills
    └── skills/                  # 14 original skills (loaded via tool-skills)
```

---

## Repository Structure (obra/superpowers)

```
superpowers/
├── .claude-plugin/         # Claude Code plugin config
│   └── plugin.json         # name, version, description
├── .codex/                 # Codex integration
├── .opencode/              # OpenCode integration
├── agents/                 # Agent definitions (need to explore)
├── commands/               # Slash commands (need to explore)
├── docs/                   # Documentation
├── hooks/                  # Lifecycle hooks (need to explore)
├── lib/                    # Shared libraries
├── skills/                 # 14 skills (main content)
└── tests/                  # Test files
```

---

## Skills Catalog (14 total)

| Skill | Purpose | Amplifier Mapping |
|-------|---------|-------------------|
| **brainstorming** | Design refinement before coding | Context file or Recipe |
| **dispatching-parallel-agents** | Concurrent subagent workflows | `delegate` tool pattern |
| **executing-plans** | Batch execution with checkpoints | Recipe |
| **finishing-a-development-branch** | Merge/PR decision workflow | Agent or Recipe |
| **receiving-code-review** | Responding to feedback | Context file |
| **requesting-code-review** | Pre-review checklist | Agent or Recipe |
| **subagent-driven-development** | Fresh agent per task + 2-stage review | Recipe with agents |
| **systematic-debugging** | 4-phase root cause process | Agent (bug-hunter analog) |
| **test-driven-development** | RED-GREEN-REFACTOR cycle | Context file (core methodology) |
| **using-git-worktrees** | Parallel development branches | Context file |
| **using-superpowers** | Introduction to skills system | Context file |
| **verification-before-completion** | Ensure it's actually fixed | Context file |
| **writing-plans** | Detailed implementation plans | Agent or Recipe |
| **writing-skills** | Create new skills (meta) | Context file |

---

## Skill File Format (Superpowers)

```markdown
---
name: skill-name
description: "Use when [triggering conditions]..."
---

# Skill Name

## Overview
Core principle in 1-2 sentences.

## When to Use
[Flowchart if decision non-obvious]
Bullet list with symptoms and use cases

## The Process
[Main content with flowcharts]

## Common Mistakes / Red Flags
What goes wrong + fixes

## Integration
References to other skills
```

**Key observations:**
1. Only 2 frontmatter fields: `name` and `description`
2. Description starts with "Use when..." (triggering conditions only!)
3. **Critical insight**: Description should NOT summarize workflow (causes AI to skip reading full content)
4. Uses Graphviz DOT flowcharts for process visualization
5. Cross-references other skills by name (e.g., `superpowers:test-driven-development`)
6. Each skill is a directory with SKILL.md + optional supporting files

---

## Skill Structure Patterns

### Discipline-Enforcing Skills (TDD, verification)
- Rules/requirements that must be followed
- Include "rationalization tables" - common excuses and rebuttals
- "Red Flags" section for self-checking
- "Iron Law" declarations

### Workflow Skills (brainstorming, executing-plans)
- Step-by-step process with flowcharts
- Integration points with other skills
- Example workflows

### Technique Skills (systematic-debugging)
- How-to guides with phases
- Application scenarios

---

## Subagent-Driven Development (Key Pattern)

This is the core orchestration pattern:

```
Per Task:
1. Dispatch implementer subagent
2. Implementer asks questions? → Answer → Continue
3. Implementer implements, tests, commits, self-reviews
4. Dispatch spec reviewer subagent
5. Spec compliant? No → Fix → Re-review
6. Dispatch code quality reviewer subagent
7. Quality approved? No → Fix → Re-review
8. Mark task complete

After all tasks:
- Dispatch final code reviewer
- Use finishing-a-development-branch
```

**Amplifier equivalent**: Recipe with agent steps:
- `implementer` agent
- `spec-reviewer` agent  
- `code-quality-reviewer` agent

---

## Key Differences: Superpowers vs Amplifier

| Aspect | Superpowers | Amplifier |
|--------|-------------|-----------|
| **Skill format** | YAML frontmatter (name, description) | YAML frontmatter (bundle: name, version) |
| **Skill loading** | Claude Code plugin system | `tool-skills` module or context files |
| **Subagents** | Claude Code subagent API | `delegate` tool |
| **Workflows** | Implicit (skills reference each other) | Explicit (Recipes with steps) |
| **Slash commands** | `/superpowers:brainstorm` | `/commands` via `tool-slash-command` |
| **Hooks** | Shell scripts | Python hook modules |

---

## Conversion Strategy

### Option A: Direct Skills Import
Use Amplifier's `tool-skills` module to load superpowers skills directly.

**Pros**: Minimal conversion, skills work as-is
**Cons**: Doesn't leverage Amplifier's recipe/agent capabilities

### Option B: Hybrid Approach (Recommended)
1. **Core methodology** (TDD, verification) → Context files
2. **Workflow processes** (subagent-driven-dev, executing-plans) → Recipes
3. **Specialized roles** (spec-reviewer, code-quality-reviewer) → Agents
4. **Reference material** → Skills via `tool-skills`

### Option C: Full Native Conversion
Convert everything to Amplifier-native patterns.

**Pros**: Full integration, leverages all Amplifier features
**Cons**: More work, may lose superpowers update compatibility

---

## Recommended Bundle Structure

```
amplifier-bundle-superpowers/
├── bundle.md                           # Root bundle (thin)
├── context/
│   ├── core-methodology.md             # TDD, verification principles
│   ├── development-workflow.md         # Overall process
│   └── skill-integration.md            # How skills work together
├── agents/
│   ├── implementer.md                  # Task implementation agent
│   ├── spec-reviewer.md                # Spec compliance reviewer
│   ├── code-quality-reviewer.md        # Code quality reviewer
│   ├── brainstormer.md                 # Design refinement agent
│   └── plan-writer.md                  # Implementation plan writer
├── recipes/
│   ├── subagent-driven-development.yaml
│   ├── brainstorm-to-plan.yaml
│   └── execute-plan.yaml
├── skills/                             # Load via tool-skills
│   ├── test-driven-development/
│   ├── systematic-debugging/
│   └── ...
└── behaviors/
    └── superpowers-workflow.yaml       # Compose agents + context
```

---

## Agents Directory (1 file)

**`agents/code-reviewer.md`** - Agent definition format:
```markdown
---
name: code-reviewer
description: |
  Use this agent when... [with examples]
model: inherit
---

[System prompt / instructions]
```

**Key insight**: Uses same `<example>` format as Amplifier agent descriptions!

---

## Commands Directory (3 files)

Slash commands that invoke skills:

| Command | Description |
|---------|-------------|
| `brainstorm.md` | Invokes `superpowers:brainstorming` skill |
| `execute-plan.md` | Invokes `superpowers:executing-plans` skill |
| `write-plan.md` | Invokes `superpowers:writing-plans` skill |

**Command format:**
```markdown
---
description: "When to use this command..."
disable-model-invocation: true
---

Invoke the superpowers:skill-name skill and follow it exactly
```

**Amplifier equivalent**: `tool-slash-command` module with `.amplifier/commands/`

---

## Hooks Directory

**`hooks.json`** - Hook configuration:
```json
{
  "hooks": {
    "SessionStart": [{
      "matcher": "startup|resume|clear|compact",
      "hooks": [{
        "type": "command",
        "command": "${CLAUDE_PLUGIN_ROOT}/hooks/session-start.sh"
      }]
    }]
  }
}
```

**`session-start.sh`** - Injects the `using-superpowers` skill into every session start.

**Amplifier equivalent**: `hook-shell` module or custom Python hook

---

## Writing-Plans Skill (Important)

Creates implementation plans with:
- **Bite-sized tasks** (2-5 minutes each)
- **Plan document header** with required sub-skill reference
- **Task structure**: Files, Steps (test→fail→implement→pass→commit)
- **Execution handoff**: Choice of subagent-driven or parallel session

**Plan location**: `docs/plans/YYYY-MM-DD-<feature-name>.md`

---

## Complete Component Mapping

| Superpowers | Format | Amplifier Equivalent |
|-------------|--------|---------------------|
| Skills | `skills/*/SKILL.md` | Skills (`tool-skills`) or Context files |
| Agents | `agents/*.md` | Agents (`agents/*.md`) |
| Commands | `commands/*.md` | Slash commands (`tool-slash-command`) |
| Hooks | `hooks/*.sh` | Hook modules or `hook-shell` |
| Plugin config | `.claude-plugin/plugin.json` | `bundle.md` frontmatter |

---

## Next Steps

1. [x] Fetch remaining skills (writing-plans, systematic-debugging, executing-plans)
2. [x] Explore agents/ and commands/ directories
3. [x] Explore hooks/ directory
4. [ ] Create bundle.md with thin pattern
5. [ ] Convert core skills to context files
6. [ ] Create recipe for subagent-driven-development
7. [ ] Define key agents (implementer, reviewers)

---

## Key Decisions Made

- **Use hybrid approach**: Core methodology as context, workflows as recipes
- **Preserve skill format**: Can load original skills via tool-skills
- **Create Amplifier-native agents**: For the reviewer/implementer pattern
- **Thin bundle pattern**: Inherit from foundation, add only superpowers-specific content

---

## Questions to Resolve

1. Should we maintain compatibility with original superpowers skills?
2. How to handle Graphviz flowcharts in Amplifier context?
3. Should slash commands be converted to agents or kept as slash commands?
