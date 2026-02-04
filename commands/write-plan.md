---
command:
  name: write-plan
  description: "Create a detailed implementation plan from a design spec or requirements"
---

# Write Plan Command

Create an implementation plan for approved designs.

Delegate to `superpowers:plan-writer` agent to create a detailed plan:

1. Read the design document or requirements
2. Break into bite-sized tasks (2-5 minutes each)
3. Include exact file paths, complete code, test commands
4. Follow TDD structure (test → fail → implement → pass → commit)
5. Save to `docs/plans/YYYY-MM-DD-<feature>.md`

**Usage:** `/write-plan [design-doc-path]`

If a design doc path is provided, create a plan from it. Otherwise, ask which design to implement.
