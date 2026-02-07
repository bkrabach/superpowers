# Bundle Comparison: obra vs microsoft amplifier-bundle-superpowers

Two independent Amplifier bundles for the [Superpowers](https://github.com/obra/superpowers) methodology exist. This report compares their approaches.

| | **obra/amplifier-bundle-superpowers** | **microsoft/amplifier-bundle-superpowers** |
|---|---|---|
| **Author** | Jesse Vincent (community) | Microsoft (ecosystem) |
| **Repo** | github.com/obra/amplifier-bundle-superpowers | github.com/microsoft/amplifier-bundle-superpowers |
| **MSFT compliance** | None (community repo) | Full (LICENSE, SECURITY, CODE_OF_CONDUCT, SUPPORT, CLA, Trademarks) |

---

## Architecture Philosophy

| Dimension | obra | microsoft |
|-----------|------|-----------|
| **Primary value** | Recipes (workflow enforcement) | Agents + Modes (interactive guidance) |
| **Approach** | Workflows as declarative YAML recipes with approval gates | Agents as delegatable specialists + modes as shortcuts |
| **User interaction** | Recipe-driven: run a recipe, approve at gates | Mode-driven: `/brainstorm`, then work interactively |
| **Autonomy model** | High autonomy between approval gates | Interactive within each mode |

**obra's philosophy**: Encode the entire workflow as recipes. The system drives the process, the human approves at gates.

**microsoft's philosophy**: Provide specialist agents and workflow modes. The human (or orchestrating agent) drives the process, delegating to specialists.

---

## Component Comparison

### Agents

| | obra | microsoft |
|---|---|---|
| **Count** | 1 | 5 |
| **Agents** | `superpowers-expert` (guide/advisor) | `brainstormer`, `plan-writer`, `implementer`, `spec-reviewer`, `code-quality-reviewer` |
| **Role** | Single expert that recommends recipes | Separate specialists for each workflow phase |

**Key difference**: obra has one "concierge" agent that guides users to the right recipe. microsoft has five role-specific agents that DO the work (implement, review, plan).

### Recipes

| | obra | microsoft |
|---|---|---|
| **Count** | 7 | 1 |
| **Recipes** | brainstorming, writing-plans, executing-plans, subagent-development, git-worktree-setup, finish-branch, full-development-cycle | subagent-driven-development |
| **Approval gates** | Yes (staged recipes with human checkpoints) | No (conditional steps only) |
| **Meta-recipe** | Yes (`full-development-cycle` composes all 6 others) | No |
| **foreach loops** | Yes (`subagent-development` iterates over tasks) | No (handles one task, repeat manually) |
| **Conditional branching** | Yes (`finish-branch` has 5-way conditional) | Yes (conditional fix steps) |

**Key difference**: obra's recipes are the core value - 7 sophisticated recipes including a meta-recipe that chains the entire workflow. microsoft has 1 simpler recipe focused on the per-task execution loop.

### Skills

| | obra | microsoft |
|---|---|---|
| **Count** | 3 (inline) | 14 (remote) |
| **Source** | Local `skills/` directory in bundle | `git+https://github.com/obra/superpowers@main#subdirectory=skills` |
| **Skills included** | test-driven-development, systematic-debugging, verification-before-completion | All 14 original Superpowers skills |
| **Loading mechanism** | Local filesystem | Remote git clone + cache |

**Key difference**: obra curated 3 skills that are true "reference knowledge" (vs workflows). microsoft pulls ALL 14 original skills from the upstream repo, always getting the latest version.

### Modes

| | obra | microsoft |
|---|---|---|
| **Count** | 0 | 3 |
| **Modes** | None | `/brainstorm`, `/write-plan`, `/execute-plan` |
| **Tool policies** | N/A | Progressive: read-only → +write_file → full access |

**Key difference**: microsoft uses modes as the primary user-facing workflow entry points. obra relies on recipes instead.

### Behaviors

| | obra | microsoft |
|---|---|---|
| **Count** | 1 | 1 |
| **Name** | `superpowers-behavior` | `superpowers-methodology-behavior` |
| **Contains** | 1 agent + 2 context files | 5 agents + 1 context file |

Both follow the same composable behavior pattern.

### Context Files

| | obra | microsoft |
|---|---|---|
| **Count** | 2 | 1 |
| **Files** | `philosophy.md` (principles + anti-patterns), `instructions.md` (operational guide + recipe catalog) | `superpowers-methodology.md` (combined principles + workflow) |
| **Approach** | Separated concerns: WHY (philosophy) vs HOW (instructions) | Combined into one document |

**Key difference**: obra separates philosophy from operational instructions. microsoft combines them. obra's `philosophy.md` includes a "Thought → Action" rationalization-catching table that's particularly well-crafted.

---

## Workflow Comparison

### The Brainstorm → Plan → Execute Pipeline

**obra's approach** (recipe-driven):
```
amplifier run "execute brainstorming recipe"
  → Recipe drives discovery, design, presents for approval
  → Human approves design at gate
  → "execute writing-plans recipe"
  → Recipe creates implementation plan, presents for approval
  → Human approves plan at gate
  → "execute subagent-development recipe"
  → foreach task: implement → spec-review → quality-review (automated)
  → Human approves completion at gate
```

**microsoft's approach** (mode-driven):
```
/brainstorm
  → Mode activates, agent guides design interactively
  → Human says "/mode off" when design is done
/write-plan
  → Mode activates, agent creates plan interactively
  → Human says "/mode off" when plan is done
/execute-plan
  → Mode activates, human triggers delegate() calls
  → Per task: implementer → spec-reviewer → code-quality-reviewer
```

**Or via meta-recipe** (obra only):
```
amplifier run "execute full-development-cycle recipe"
  → Runs ALL stages with 3 approval gates
  → Brainstorm → Approve → Plan → Approve → Implement → Approve → Finish
```

### Two-Stage Review Pattern

Both implement the same review pattern, but differently:

| Aspect | obra | microsoft |
|--------|------|-----------|
| **Mechanism** | Recipe steps with `foreach` loops | Agent delegation in mode/recipe |
| **Iteration** | Recipe loops until reviewer approves | Conditional fix steps |
| **Review agents** | Uses `foundation:zen-architect` (REVIEW mode) | Has dedicated `spec-reviewer` and `code-quality-reviewer` agents |

---

## Strengths of Each Approach

### obra Strengths

1. **Recipe depth** - 7 recipes covering the ENTIRE development lifecycle including git worktree setup and branch finishing
2. **Meta-recipe** - The `full-development-cycle.yaml` (444 lines) chains everything into one executable workflow with 3 approval gates
3. **Approval gates** - Human checkpoints are first-class citizens with `default: deny`
4. **foreach iteration** - Properly iterates over tasks with result collection
5. **Philosophy separation** - Clean split between WHY (philosophy.md) and HOW (instructions.md)
6. **Finish workflow** - 5-way conditional (MERGE/PR/KEEP/DISCARD/BLOCKED-MERGE) for branch completion
7. **Git worktree support** - Automated isolated workspace setup with project type detection
8. **Analysis documentation** - `ANALYSIS_SUMMARY.md` documents the porting decisions

### microsoft Strengths

1. **5 dedicated agents** - Each workflow role has a purpose-built agent with clear scope boundaries
2. **Mode shortcuts** - `/brainstorm`, `/write-plan`, `/execute-plan` are instantly accessible
3. **Tool permission escalation** - Modes progressively unlock tools (read-only → write → full)
4. **Full skills library** - All 14 original skills loaded from upstream, always latest
5. **Remote skill fetching** - No submodule, auto-cached from GitHub
6. **MSFT compliance** - Ready for Microsoft open-source ecosystem
7. **Agent scope boundaries** - Each agent has explicit "What you DO / DON'T check" sections
8. **Composable behavior** - Easy to include just the methodology in another bundle

---

## Feature Gap Analysis

### obra has, microsoft doesn't

| Feature | Impact | Effort to Add |
|---------|--------|---------------|
| Meta-recipe (full development cycle) | High - enables end-to-end autonomous workflow | Medium - compose existing recipe |
| Approval gates in recipes | High - human checkpoints prevent runaway execution | Low - convert to staged recipe |
| foreach task iteration | High - handles multi-task plans automatically | Low - add foreach to recipe |
| Git worktree setup recipe | Medium - automated isolated workspace | Medium - new recipe |
| Branch finishing recipe | Medium - structured merge/PR workflow | Medium - new recipe |
| Executing-plans recipe (batch mode) | Medium - alternative to subagent pattern | Medium - new recipe |
| Separated philosophy/instructions context | Low - cleaner conceptual split | Low - split context file |

### microsoft has, obra doesn't

| Feature | Impact | Effort to Add |
|---------|--------|---------------|
| Modes (/brainstorm, /write-plan, /execute-plan) | High - instant workflow entry | Low - create mode files |
| Tool permission escalation | Medium - prevents accidental writes during design | Low - mode tool policies |
| 5 role-specific agents | Medium - better agent delegation patterns | Medium - create agent files |
| Remote skills loading (all 14) | Medium - always latest, no submodule | Low - change skills config |
| MSFT compliance files | Required for Microsoft org | Low - copy templates |
| Dedicated spec-reviewer agent | Medium - clear review scope | Low - create agent |
| Dedicated code-quality-reviewer agent | Medium - clear quality scope | Low - create agent |

---

## Summary

These bundles represent two valid but distinct approaches to the same problem:

- **obra's bundle** is **recipe-centric** - the workflow IS the recipes. Sophisticated YAML workflows with approval gates drive the entire process. One expert agent guides users to the right recipe. Best for users who want structured, automated workflows with human checkpoints.

- **microsoft's bundle** is **agent-centric** - the workflow is enabled by specialist agents and modes. Users enter a mode, interact with agents, and drive the process. Best for users who want interactive guidance with the flexibility to adapt on the fly.

The ideal bundle would combine both: obra's recipe sophistication (especially the meta-recipe, foreach loops, and approval gates) with microsoft's agent specialization, mode shortcuts, and remote skills loading.
