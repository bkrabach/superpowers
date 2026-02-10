# Superpowers Bundle Fidelity Audit

## How Well Does microsoft/amplifier-bundle-superpowers Capture obra/superpowers?

Three independent analyses were run in parallel covering skill-by-skill content fidelity, anti-rationalization enforcement depth, and recipe workflow structure.

---

## Overall Scorecard

| Skill | Fidelity | Representation |
|-------|----------|----------------|
| brainstorming | **FULL** | Mode + Agent + Recipe (54→541 lines) |
| writing-plans | **FULL** | Mode + Agent + Recipe (116→683 lines) |
| subagent-driven-development | **FULL** | Mode + 3 Agents + Recipe (401→1063 lines) |
| test-driven-development | **PARTIAL** | Context files + Implementer agent (670→~216 embedded) |
| systematic-debugging | **PARTIAL** | Mode + Context (702→266) |
| verification-before-completion | **FULL** | Mode (139→195 lines) |
| finishing-a-development-branch | **FULL** | Mode + Recipe (200→668 lines) |
| using-superpowers | **FULL** | Standing order + Instructions (87→327 lines) |
| executing-plans | **FULL** | Recipe |
| using-git-worktrees | **FULL** | Recipe |
| dispatching-parallel-agents | **PARTIAL** | No explicit parallel dispatch guidance |
| receiving-code-review | **PARTIAL** | Split across two reviewer agents |
| requesting-code-review | **PARTIAL** | Split across two reviewer agents |
| writing-skills | **MISSING** | No Amplifier representation |

**Recipe fidelity: 7/7 FAITHFUL** - All recipes structurally preserved with correct agent remapping.

---

## Where We're BETTER Than the Original

1. **Tool enforcement** - Mode tool policies structurally prevent violations. The original can only prompt-engineer compliance; we can BLOCK write_file/edit_file. This is an architectural advantage the original cannot replicate.

2. **Approval gates in recipes** - Formal human checkpoints with `default: deny`. The original's "say Ready to continue" is fragile; our recipe gates are structural.

3. **Two-stage review as dedicated agents** - The original uses one `code-reviewer` skill. Our separation into `spec-reviewer` + `code-quality-reviewer` with dedicated agents is cleaner and more focused.

4. **Hybrid orchestrator/agent pattern** - Conversation stays with the user (interactive), artifacts are created by focused agents (clean context). The original has the main agent do everything.

5. **Recipe autopilot** - `full-development-cycle.yaml` meta-recipe chains the entire pipeline with 3 approval gates. Structurally enforced transitions that prompt engineering can't match.

6. **Always-latest skills** - Remote git source fetches from `obra/superpowers@main`, so users always get Jesse's latest skills without manual updates.

7. **Mode transition guidance** - Each mode has explicit golden path and dynamic transition suggestions. The original's skill-to-skill references are implicit.

---

## Where We're WEAKER (Gaps to Close)

### P0: TDD Anti-Patterns Not First-Class (HIGH IMPACT)

The original has **670 lines** of TDD content across two files:
- `test-driven-development/SKILL.md` (371 lines) - 4 Good/Bad code examples, verification checklists, "When Stuck" troubleshooting, "Why Order Matters" persuasion
- `testing-anti-patterns.md` (299 lines) - 5 anti-patterns each with gate functions

Our bundle embeds ~216 lines of TDD content across context files and the implementer agent. The iron laws are there, but the **depth** is not. Most critically:

- **Missing: 4 paired Good/Bad code examples** that make TDD concrete
- **Missing: 5 testing anti-patterns with gate functions** (testing mock behavior, test-only methods, mocking without understanding, incomplete mocks, integration tests as afterthought)
- **Missing: "When Stuck" troubleshooting table** (4 entries)
- **Missing: "Why Order Matters" persuasion section** (5 detailed counter-arguments)
- **Missing: Verification checklist** (8-item TDD compliance check)

**The implementer agent - the one that actually writes code - has ~35% of the enforcement text the original carries.** This is the critical asymmetry: enforcement is strongest at the orchestrator level but weakest where code actually gets written.

**Fix:** Enrich `agents/implementer.md` with anti-rationalization content, or create `context/tdd-deep-dive.md` that the implementer loads.

### P1: Debugging Companion Techniques Not First-Class (MEDIUM IMPACT)

The original has 3 companion documents totaling ~400 lines:
- `root-cause-tracing.md` (169 lines) - backward tracing technique with code examples
- `defense-in-depth.md` (122 lines) - four-layer validation pattern
- `condition-based-waiting.md` (115 lines) - flaky test fixing with `waitFor` implementation
- Plus `find-polluter.sh` - test bisection script

These are available via `load_skill` but not embedded in the debug mode.

**Fix:** Reference these explicitly in debug mode, or create `context/debugging-techniques.md`.

### P2: Spec Reviewer Lacks "Suspicion" Tone (LOW IMPACT)

The original's `spec-reviewer-prompt.md` says: "CRITICAL: Do Not Trust the Report. The implementer finished suspiciously quickly. Their report may be incomplete, inaccurate, or optimistic."

Our spec-reviewer is professional but lacks this adversarial posture that drives more thorough reviews.

**Fix:** Add distrust framing to `agents/spec-reviewer.md`.

### P3: "Human Partner Signals" Missing from Debug Mode (LOW IMPACT)

The original debugging skill has a "Human Partner's Signals" section with 5 correction signals and how to respond. This human collaboration pattern isn't in our debug mode.

### P4: Anti-Rationalization Depth Gap

| Location | Original Entries | Our Entries | Coverage |
|----------|-----------------|-------------|----------|
| using-superpowers (→ instructions.md) | 12 | 11 | 92% |
| test-driven-development (→ methodology.md) | 11 | 6 | 55% |
| systematic-debugging (→ debug mode) | 8+2 | 10 | 100% |
| verification-before-completion (→ verify mode) | 8 | 10 | 125% |
| **Total** | **41** | **37** | **90%** |

The gap is concentrated in TDD enforcement (5 missing entries).

---

## Structural Wins Unique to Amplifier

These are things our bundle does that the original CANNOT:

| Capability | Original | Our Bundle |
|-----------|----------|-----------|
| Block write tools during brainstorm | Impossible (prompt only) | Mode tool policy enforces it |
| Block write tools during planning | Impossible | Mode tool policy enforces it |
| Force delegation in execution | Prompt engineering | write_file/edit_file blocked structurally |
| Formal approval gates | "Say Ready to continue" | Recipe approval with `default: deny` |
| Auto-fetch latest skills | Manual plugin update | `git+https://...@main` always latest |
| Composable methodology | All-or-nothing | Behavior YAML for partial inclusion |
| Agent tool scoping | All agents same tools | Each agent declares its own tools |

---

## Priority Fix List

| Priority | Fix | Impact | Effort |
|----------|-----|--------|--------|
| **P0** | Enrich implementer agent with TDD depth (anti-patterns, examples, checklist) | HIGH - fixes the critical enforcement asymmetry | Medium |
| **P1** | Create debugging techniques context or explicit skill references in debug mode | MEDIUM - makes companion techniques discoverable | Low |
| **P2** | Add "suspicion" framing to spec-reviewer | LOW - drives more thorough reviews | Low |
| **P3** | Add human partner signals to debug mode | LOW - improves collaboration | Low |
| **P4** | Complete the 5 missing TDD anti-rationalization entries | LOW - already at 90% coverage | Low |

---

## Conclusion

The bundle achieves **FULL fidelity on 8 of 14 skills** and **PARTIAL on 5** (with 1 MISSING - the meta-skill `writing-skills` which is about creating new skills, not a workflow).

The partial ratings are concentrated in two areas:
1. **TDD depth** - Iron laws are there but the persuasive depth, code examples, and anti-patterns aren't first-class
2. **Debugging companions** - Core 4-phase process is faithful but the technique companion docs aren't embedded

Both gaps are mitigated by the `skills.sources` passthrough that makes the full original skills available via `load_skill`. The question is whether "available on demand" is sufficient or whether they need to be "always present."

The bundle's **structural enforcement** (tool stripping, approval gates, agent tool scoping) gives it capabilities the original simply cannot achieve through prompt engineering alone. This is the Amplifier advantage.
