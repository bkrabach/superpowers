---
command:
  name: execute-plan
  description: "Execute an implementation plan using subagent-driven development"
---

# Execute Plan Command

Execute an implementation plan with subagent-driven development.

This uses the superpowers workflow:
1. For each task in the plan:
   - Implementer agent implements following TDD
   - Spec reviewer validates against specification
   - Code quality reviewer checks quality
2. Final review of entire implementation

**Usage:** `/execute-plan [plan-path]`

If a plan path is provided, execute that plan. Otherwise, look for recent plans in `docs/plans/`.

**Execution modes:**
- **Subagent-Driven** (default): Fresh agent per task, two-stage review, fast iteration
- **Manual**: Step through tasks with human checkpoints

Ask which mode if not specified.
