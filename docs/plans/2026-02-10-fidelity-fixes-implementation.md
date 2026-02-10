# Implementation Plan: Fidelity Fixes for amplifier-bundle-superpowers

**Date:** 2026-02-10
**Source:** [Fidelity Audit Report](../fidelity-audit-report.md)
**Target repo:** `/home/bkrabach/repos/superpowers/amplifier-bundle-superpowers`

---

## Architectural Decisions

Before the task list, three structural decisions that shape everything below.

### Decision 1: TDD Depth Goes in a Context File, Not Inline in the Implementer

**The problem:** The implementer agent is 131 lines. The missing TDD content (anti-patterns with gate functions, Good/Bad examples, verification checklist, "When Stuck" table, "Why Order Matters" rebuttals) totals ~350 lines. Inlining would bloat the agent to ~480 lines — almost 4x the size of any other agent in the bundle.

**The decision:** Create `context/tdd-depth.md` and have the implementer `@mention` it.

**Why this is right:**
- **Separation of concerns.** The implementer agent defines *process* (what to do step by step). The TDD depth content is *reference material* (what to watch for, why it matters, how to recognize anti-patterns). These are different concerns.
- **Reusability.** Other agents and modes can also @mention the same context if needed — the debug mode references TDD for bug-fix testing, and the code-quality-reviewer needs to recognize testing anti-patterns too.
- **Amplifier pattern alignment.** The existing bundle already uses `context/philosophy.md` and `context/superpowers-methodology.md` as shared reference material @mentioned by the bundle root. This follows the same pattern.
- **Token budget.** A context file loads into the agent's context at invocation time, same as inline content. No functional difference to the model — it sees all of it. But it keeps the agent file focused and readable for humans.

### Decision 2: Debugging Techniques Go in a Context File, Not Inline in Debug Mode

**The problem:** The debug mode is 266 lines. The three companion techniques (root-cause-tracing, defense-in-depth, condition-based-waiting) total ~400 lines. Inlining would make a 666-line mode file.

**The decision:** Create `context/debugging-techniques.md` and have the debug mode `@mention` it.

**Why this is right:**
- **Same reasoning as Decision 1.** The debug mode defines the *4-phase process*. The companion techniques are *reference material* for specific situations within that process.
- **Selective relevance.** Not every debugging session needs all three techniques. Root-cause-tracing applies to Phase 1 (investigation). Defense-in-depth applies to Phase 4 (fix design). Condition-based-waiting applies only to flaky tests. Loading them as reference is appropriate — they inform the process without dominating it.
- **The mode already references skills.** The debug mode has `load_skill` in its tool list and the bundle has `skills.sources` pointing to obra/superpowers. The context file provides a curated summary with explicit phase-mapping that raw skills don't have.

### Decision 3: Spec Reviewer Suspicion and Debug Human Signals Go Inline

**The problem:** The spec-reviewer needs ~20 lines of "suspicion" framing. The debug mode needs ~15 lines of human partner signals.

**The decision:** Add these directly to their respective files.

**Why this is right:**
- **Small additions.** Neither exceeds 25 lines. No file size concern.
- **Behavioral, not reference.** The suspicion framing changes HOW the spec-reviewer operates — it's a posture, not reference material. It belongs in the agent's core instructions. Similarly, human partner signals are about the debugging PROCESS itself, not a technique to look up.

---

## Task List

### P0: Enrich Implementer Agent with TDD Depth

#### Task 1: Create `context/tdd-depth.md` with Testing Anti-Patterns

**File:** `context/tdd-depth.md` (NEW)

**What to write:** A context file containing the 5 testing anti-patterns from `testing-anti-patterns.md`, each with:
- The violation (Bad code example)
- Why it's wrong (2-3 bullet points)
- The fix (Good code example)
- The gate function (decision tree to prevent the anti-pattern)

**Exact content to port (from `testing-anti-patterns.md`):**
1. Anti-Pattern 1: Testing Mock Behavior (lines 22-61) — asserting on mock elements instead of real behavior, with gate function
2. Anti-Pattern 2: Test-Only Methods in Production (lines 63-116) — adding destroy()/cleanup methods to production classes, with gate function
3. Anti-Pattern 3: Mocking Without Understanding (lines 118-175) — over-mocking that breaks test logic, with gate function
4. Anti-Pattern 4: Incomplete Mocks (lines 177-226) — partial mocks that hide structural assumptions, with gate function
5. Anti-Pattern 5: Integration Tests as Afterthought (lines 228-249) — claiming "ready for testing" after implementation

Also port the Quick Reference table (lines 275-282) and Red Flags list (lines 284-291).

**Acceptance criteria:**
- [ ] File exists at `context/tdd-depth.md`
- [ ] All 5 anti-patterns present with Bad/Good code examples
- [ ] All 5 gate functions present (the BEFORE/ASK/IF decision trees)
- [ ] Quick Reference table and Red Flags list present
- [ ] Code examples are TypeScript (matching originals)
- [ ] File opens with a header explaining its purpose: reference material for TDD enforcement

---

#### Task 2: Add "When Stuck" Troubleshooting Table to `context/tdd-depth.md`

**File:** `context/tdd-depth.md` (append)

**What to write:** The 4-entry troubleshooting table from `SKILL.md` lines 343-350:

```markdown
## When Stuck

| Problem | Solution |
|---------|----------|
| Don't know how to test | Write wished-for API. Write assertion first. Ask your human partner. |
| Test too complicated | Design too complicated. Simplify interface. |
| Must mock everything | Code too coupled. Use dependency injection. |
| Test setup huge | Extract helpers. Still complex? Simplify design. |
```

**Acceptance criteria:**
- [ ] Table has exactly 4 entries matching the original
- [ ] Placed after the anti-patterns section
- [ ] "Ask your human partner" phrasing preserved (not "ask the user")

---

#### Task 3: Add "Why Order Matters" Rebuttals to `context/tdd-depth.md`

**File:** `context/tdd-depth.md` (append)

**What to write:** The 5 extended rebuttals from `SKILL.md` lines 206-254. These are the detailed counter-arguments, not the table summaries (those are already in `superpowers-methodology.md`). Each rebuttal is a heading with 2-4 paragraphs of persuasive text:

1. **"I'll write tests after to verify it works"** — Tests written after pass immediately. Passing immediately proves nothing. You never saw it catch the bug. (lines 208-217)
2. **"I already manually tested all the edge cases"** — Manual testing is ad-hoc. No record. Can't re-run. "It worked when I tried it" ≠ comprehensive. (lines 219-227)
3. **"Deleting X hours of work is wasteful"** — Sunk cost fallacy. The time is already gone. Working code without real tests is technical debt. (lines 229-235)
4. **"TDD is dogmatic, being pragmatic means adapting"** — TDD IS pragmatic. Finds bugs before commit. "Pragmatic" shortcuts = debugging in production = slower. (lines 237-245)
5. **"Tests after achieve the same goals - it's spirit not ritual"** — Tests-after answer "What does this do?" Tests-first answer "What should this do?" 30 minutes of tests after ≠ TDD. (lines 247-254)

**Acceptance criteria:**
- [ ] All 5 rebuttals present with full argumentative text
- [ ] Each rebuttal has a quoted heading matching the rationalization it counters
- [ ] Persuasive depth preserved — not summarized into table rows
- [ ] Section header: "## Why Order Matters"

---

#### Task 4: Add Good/Bad Paired Code Examples to `context/tdd-depth.md`

**File:** `context/tdd-depth.md` (append)

**What to write:** At least 2 Good/Bad paired code examples from `SKILL.md`. Port these two pairs:

**Pair 1: RED phase** (lines 75-106) — Writing a failing test
- Good: `test('retries failed operations 3 times')` — Clear name, tests real behavior, one thing
- Bad: `test('retry works')` — Vague name, tests mock not code

**Pair 2: GREEN phase** (lines 134-164) — Minimal implementation
- Good: Simple `retryOperation` with for loop — Just enough to pass
- Bad: `retryOperation` with options object (maxRetries, backoff, onRetry) — Over-engineered/YAGNI

**Acceptance criteria:**
- [ ] Both pairs present with `<Good>` and `<Bad>` annotations (or equivalent clear labeling)
- [ ] Each pair has a brief explanation of why one is good and the other bad
- [ ] Code is TypeScript matching the originals
- [ ] Placed under a "## Good Tests vs Bad Tests" section

---

#### Task 5: Add 8-Item TDD Verification Checklist to `context/tdd-depth.md`

**File:** `context/tdd-depth.md` (append)

**What to write:** The verification checklist from `SKILL.md` lines 329-340:

```markdown
## Verification Checklist

Before marking work complete:

- [ ] Every new function/method has a test
- [ ] Watched each test fail before implementing
- [ ] Each test failed for expected reason (feature missing, not typo)
- [ ] Wrote minimal code to pass each test
- [ ] All tests pass
- [ ] Output pristine (no errors, warnings)
- [ ] Tests use real code (mocks only if unavoidable)
- [ ] Edge cases and errors covered

Can't check all boxes? You skipped TDD. Start over.
```

**Acceptance criteria:**
- [ ] Exactly 8 checklist items
- [ ] "Can't check all boxes? You skipped TDD. Start over." closing line present
- [ ] Uses checkbox markdown format `- [ ]`

---

#### Task 6: Add 5 Missing Anti-Rationalization Entries to `context/tdd-depth.md`

**File:** `context/tdd-depth.md` (append)

**What to write:** The 5 entries from the original `SKILL.md` Common Rationalizations table (lines 256-270) that are NOT already in `context/superpowers-methodology.md`. Cross-reference to identify the missing ones.

Already in `superpowers-methodology.md`: "Too simple to test", "I'll test after", "Already manually tested", "Deleting X hours is wasteful", "Need to explore first", "TDD will slow me down" (6 entries).

**Missing from the original's 11-entry table:**
1. **"Tests after achieve same goals"** — "Tests-after = 'what does this do?' Tests-first = 'what should this do?'"
2. **"Keep as reference, write tests first"** — "You'll adapt it. That's testing after. Delete means delete."
3. **"Test hard = design unclear"** — "Listen to test. Hard to test = hard to use."
4. **"Manual test faster"** — "Manual doesn't prove edge cases. You'll re-test every change."
5. **"Existing code has no tests"** — "You're improving it. Add tests for existing code."

**Acceptance criteria:**
- [ ] All 5 missing entries present in a "## Common Rationalizations" table
- [ ] Each entry has Excuse and Reality columns matching the original
- [ ] No duplication with entries already in `superpowers-methodology.md`

---

#### Task 7: Wire `context/tdd-depth.md` into the Implementer Agent

**File:** `agents/implementer.md`

**What to change:** Add an `@mention` reference to load the TDD depth context. Add it after the existing agent instructions, before the "Output Format" section.

**Exact edit:** After the "Iron Laws" section (around line 97), add:

```markdown
## TDD Reference

For detailed anti-patterns, gate functions, code examples, and troubleshooting:

@superpowers:context/tdd-depth.md
```

Also add a brief inline reminder connecting the reference material to the process. After the self-review checklist (line 81), add one line:

```markdown
If you can't check all boxes, consult the TDD reference below for what went wrong.
```

**Acceptance criteria:**
- [ ] `@superpowers:context/tdd-depth.md` appears in `agents/implementer.md`
- [ ] The @mention is placed so the agent sees it as reference material, not process instructions
- [ ] The self-review checklist links to the reference
- [ ] No other structural changes to the implementer agent's process sections

---

### P1: Reference Debugging Companion Techniques in Debug Mode

#### Task 8: Create `context/debugging-techniques.md` with Root-Cause Tracing

**File:** `context/debugging-techniques.md` (NEW)

**What to write:** A context file opening with a purpose header, then the root-cause tracing technique from `root-cause-tracing.md`. Port:

1. The overview and core principle (line 7-8): "Trace backward through the call chain until you find the original trigger, then fix at the source."
2. The 5-step tracing process (lines 34-65): Observe Symptom → Find Immediate Cause → Ask "What Called This?" → Keep Tracing Up → Find Original Trigger
3. The "Adding Stack Traces" instrumentation section (lines 67-96)
4. The real example (lines 109-128): Empty projectDir traced through 5 levels
5. The key principle (line 154): "NEVER fix just where the error appears."
6. Stack Trace Tips (lines 157-161)

**Do NOT port:** The graphviz diagrams (they don't render in agent context) or the `find-polluter.sh` reference (not available in the bundle). Replace the diagram with equivalent text.

**Acceptance criteria:**
- [ ] File exists at `context/debugging-techniques.md`
- [ ] 5-step tracing process present with numbered steps
- [ ] Stack trace instrumentation code example present (TypeScript)
- [ ] Real-world example present showing the 5-level trace chain
- [ ] "NEVER fix just where the error appears" principle prominently stated
- [ ] No graphviz blocks (replaced with text equivalents)

---

#### Task 9: Add Defense-in-Depth to `context/debugging-techniques.md`

**File:** `context/debugging-techniques.md` (append)

**What to write:** The four-layer validation pattern from `defense-in-depth.md`. Port:

1. The core principle (line 7): "Validate at EVERY layer data passes through. Make the bug structurally impossible."
2. The framing (lines 11-12): Single validation = "We fixed the bug." Multiple layers = "We made the bug impossible."
3. All four layers with code examples:
   - Layer 1: Entry Point Validation (lines 23-37)
   - Layer 2: Business Logic Validation (lines 40-49)
   - Layer 3: Environment Guards (lines 52-69)
   - Layer 4: Debug Instrumentation (lines 72-84)
4. The "Applying the Pattern" 4-step process (lines 89-94)
5. The key insight (lines 114-122): All four layers were necessary, each caught bugs the others missed

**Acceptance criteria:**
- [ ] All 4 layers present with TypeScript code examples
- [ ] "We fixed the bug" vs "We made the bug impossible" framing present
- [ ] 4-step application process present
- [ ] Key insight about layers catching different bugs present

---

#### Task 10: Add Condition-Based Waiting to `context/debugging-techniques.md`

**File:** `context/debugging-techniques.md` (append)

**What to write:** The flaky test fixing technique from `condition-based-waiting.md`. Port:

1. Core principle (line 7): "Wait for the actual condition you care about, not a guess about how long it takes."
2. The Before/After pattern (lines 36-46): `setTimeout` vs `waitFor`
3. Quick Patterns table (lines 50-56): 5 scenarios with their `waitFor` patterns
4. The `waitFor` implementation (lines 62-79): Generic polling function
5. Common Mistakes (lines 85-93): 3 mistakes with fixes
6. When arbitrary timeout IS correct (lines 96-108): The documented-and-justified exception
7. Real-world impact stats (lines 110-115): 15 flaky tests fixed, 60%→100% pass rate

**Acceptance criteria:**
- [ ] Before/After code comparison present
- [ ] Quick Patterns table with 5 entries present
- [ ] `waitFor` implementation code present
- [ ] 3 common mistakes with fixes present
- [ ] "When arbitrary timeout IS correct" exception documented
- [ ] Real-world impact stats present

---

#### Task 11: Add Phase-Mapping Header to `context/debugging-techniques.md`

**File:** `context/debugging-techniques.md` (edit the top)

**What to write:** After writing all three technique sections (Tasks 8-10), add a phase-mapping table at the top of the file that connects each technique to the debug mode's 4 phases:

```markdown
# Debugging Companion Techniques

Reference material for the 4-phase systematic debugging process. Each technique maps to specific phases where it's most useful.

| Technique | Primary Phase | When to Reach For It |
|-----------|--------------|---------------------|
| Root-Cause Tracing | Phase 1 (Investigate) | Error appears deep in call stack, unclear where bad data originated |
| Defense-in-Depth | Phase 4 (Fix) | Designing the fix — add validation at every layer, not just one |
| Condition-Based Waiting | Phase 1 + Phase 4 | Flaky tests, race conditions, timing-dependent failures |
```

**Acceptance criteria:**
- [ ] Phase-mapping table is the FIRST content after the file header
- [ ] All 3 techniques mapped to their primary phases
- [ ] "When to Reach For It" column gives concrete trigger conditions
- [ ] Table references Phase numbers matching the debug mode (1-4)

---

#### Task 12: Wire `context/debugging-techniques.md` into Debug Mode

**File:** `modes/debug.md`

**What to change:** Add an `@mention` to load the debugging techniques context. Place it after the "The Four Phases" section and before the "Red Flags" section, so the techniques are available as reference during investigation.

**Exact edit:** After Phase 4's content (around line 198, after "Use the bug-hunter's findings to inform your Phase 3 hypothesis."), add:

```markdown
## Companion Techniques

For specific debugging situations, consult these techniques:

@superpowers:context/debugging-techniques.md
```

Also add brief inline references within the relevant phases to point to the techniques:

In Phase 1, step 5 "Trace Data Flow" (around line 103), after "Fix at source, not at symptom", add:
```
   See **Root-Cause Tracing** in companion techniques for the full backward-tracing method.
```

In Phase 4 (around line 156), after "Now delegate the fix.", add:
```
   When designing the fix, consider **Defense-in-Depth** — validate at every layer, not just one. See companion techniques.
```

**Acceptance criteria:**
- [ ] `@superpowers:context/debugging-techniques.md` appears in `modes/debug.md`
- [ ] Phase 1 has an inline reference to root-cause tracing
- [ ] Phase 4 has an inline reference to defense-in-depth
- [ ] No structural changes to the 4-phase process itself
- [ ] The @mention is placed between the phases and the red flags sections

---

### P2: Add "Suspicion" Framing to Spec Reviewer

#### Task 13: Add Adversarial Posture to Spec Reviewer Agent

**File:** `agents/spec-reviewer.md`

**What to change:** Add the "Do Not Trust the Report" framing from `spec-reviewer-prompt.md` lines 21-36. Insert it after "## Your Mandate" section (line 33) and before "## Review Process" (line 41).

**Exact content to add:**

```markdown
## CRITICAL: Do Not Trust the Report

The implementer's report may be incomplete, inaccurate, or optimistic. You MUST verify everything independently.

**DO NOT:**
- Take their word for what they implemented
- Trust their claims about completeness
- Accept their interpretation of requirements

**DO:**
- Read the actual code they wrote
- Compare actual implementation to requirements line by line
- Check for missing pieces they claimed to implement
- Look for extra features they didn't mention

**Verify by reading code, not by trusting report.**
```

**Acceptance criteria:**
- [ ] "CRITICAL: Do Not Trust the Report" section present in `agents/spec-reviewer.md`
- [ ] Placed between "Your Mandate" and "Review Process" sections
- [ ] Contains both DO NOT and DO lists
- [ ] "Verify by reading code, not by trusting report" closing line present
- [ ] Does NOT include the "finished suspiciously quickly" phrasing (too specific to the original's dispatch context — our agent is persistent, not dispatched per-task with a prompt template)

**Design note:** The original says "The implementer finished suspiciously quickly." This framing makes sense in the original's dispatch-a-subagent-with-a-prompt model where you're literally telling the reviewer "the agent just returned fast." In our bundle, the spec-reviewer is a dedicated agent invoked after the implementer completes — the "suspiciously quickly" framing doesn't apply. We keep the adversarial posture (don't trust the report) but drop the specific timing claim.

---

### P3: Add Human Partner Signals to Debug Mode

#### Task 14: Add Human Partner Signals Section to Debug Mode

**File:** `modes/debug.md`

**What to write:** The original `systematic-debugging/SKILL.md` has a "Human Partner's Signals" section with correction patterns. We need to port the concept adapted to the Amplifier context (where the human is the user, not "Jesse").

Add after the "Anti-Rationalization Table" (around line 231) and before "Quick Reference":

```markdown
## Human Partner Signals

When the user gives correction signals during debugging, listen carefully:

| Signal | Meaning | Your Response |
|--------|---------|---------------|
| "Are we testing the behavior of a mock?" | You're asserting on mock existence, not real behavior | Stop. Remove mock assertion. Test real component. |
| "Do we need to be using a mock here?" | Your mocks are too complex — consider integration test | Evaluate if real component is simpler than the mock |
| "What does this test actually prove?" | Test may be trivial or testing the wrong thing | Articulate what behavior the test verifies. If you can't, rewrite it. |
| "Have you traced this back to the source?" | You're fixing at the symptom, not the root cause | Return to Phase 1. Trace backward through the call chain. |
| "How many layers validate this?" | Single-point fix is fragile | Apply defense-in-depth: validate at entry, business logic, environment, and debug layers. |
```

**Acceptance criteria:**
- [ ] "Human Partner Signals" section present in `modes/debug.md`
- [ ] Table has 5 signal entries
- [ ] Each signal has a clear meaning and required response
- [ ] Uses "user" not "human partner" or "Jesse" (Amplifier convention)
- [ ] References connect back to techniques (defense-in-depth, root-cause tracing)
- [ ] Placed after anti-rationalization table, before quick reference

---

### P4: Complete Missing TDD Anti-Rationalization Entries

#### Task 15: Add Missing Entries to `context/superpowers-methodology.md`

**File:** `context/superpowers-methodology.md`

**What to change:** The "Common Rationalizations" table (lines 45-52) has 6 entries. Add the 5 missing entries from the original to bring it to 11 (matching the original's coverage).

**Entries to add to the existing table:**

```markdown
| "Tests after achieve same goals" | Tests-after = "what does this do?" Tests-first = "what should this do?" |
| "Keep as reference, write tests first" | You'll adapt it. That's testing after. Delete means delete. |
| "Test hard = design unclear" | Listen to test. Hard to test = hard to use. |
| "Manual test faster" | Manual doesn't prove edge cases. You'll re-test every change. |
| "Existing code has no tests" | You're improving it. Add tests for existing code. |
```

**Also add to the Red Flags list (lines 56-63)** the matching entries:
```markdown
- Keeping code as "reference" while writing tests
- Saying "tests after achieve the same purpose"
```

**Acceptance criteria:**
- [ ] Common Rationalizations table has 11 entries (6 existing + 5 new)
- [ ] Red Flags list has 8 entries (6 existing + 2 new)
- [ ] No duplicate entries
- [ ] New entries match the original's Excuse/Reality wording exactly
- [ ] Table formatting consistent with existing entries

---

## Verification Plan

After all 15 tasks are complete, verify the following:

### Content Verification
1. **Word count check:** `context/tdd-depth.md` should be ~300-400 lines (covering anti-patterns, gate functions, examples, checklist, rebuttals, rationalizations)
2. **Word count check:** `context/debugging-techniques.md` should be ~250-350 lines (covering 3 techniques with code examples and phase mapping)
3. **Cross-reference check:** Every anti-rationalization entry in `superpowers-methodology.md` exists in either `tdd-depth.md` or `philosophy.md` (no orphaned entries)
4. **@mention resolution check:** Verify `@superpowers:context/tdd-depth.md` resolves correctly from the implementer agent
5. **@mention resolution check:** Verify `@superpowers:context/debugging-techniques.md` resolves correctly from debug mode

### Fidelity Verification
Run the audit metrics again after implementation:

| Metric | Before | Target After |
|--------|--------|-------------|
| TDD content coverage | ~35% | ~90% |
| Testing anti-patterns | 0/5 | 5/5 |
| Gate functions | 0/5 | 5/5 |
| "When Stuck" table | missing | present (4 entries) |
| "Why Order Matters" rebuttals | missing | present (5 rebuttals) |
| Good/Bad code examples | 0 | 2+ pairs |
| Verification checklist | missing | present (8 items) |
| Anti-rationalization coverage | 37/41 (90%) | 41/41 (100%) |
| Debugging companion techniques | 0/3 referenced | 3/3 referenced with phase mapping |
| Spec reviewer suspicion framing | absent | present |
| Human partner signals | absent | present (5 signals) |

### Structural Verification
- [ ] No file in `agents/` exceeds 200 lines (agents stay focused)
- [ ] No file in `modes/` exceeds 350 lines (modes stay readable)
- [ ] New context files follow the existing pattern (`context/*.md`)
- [ ] All @mentions use `@superpowers:context/` prefix (bundle-relative paths)
- [ ] No content duplication between `tdd-depth.md` and `superpowers-methodology.md` (the methodology has summary tables; the depth file has extended arguments and code examples)

---

## Task Dependency Graph

```
Task 1 (anti-patterns)
Task 2 (when stuck)         ──┐
Task 3 (why order matters)    ├── All append to tdd-depth.md
Task 4 (good/bad examples)   │
Task 5 (checklist)            │
Task 6 (rationalizations)   ──┘
                               │
Task 7 (wire into implementer) ◄── depends on Tasks 1-6

Task 8  (root-cause tracing)
Task 9  (defense-in-depth)    ├── All build debugging-techniques.md
Task 10 (condition-based)     │
Task 11 (phase mapping)      ──┘
                               │
Task 12 (wire into debug)     ◄── depends on Tasks 8-11

Task 13 (spec reviewer suspicion)  ── independent
Task 14 (human partner signals)    ── independent
Task 15 (methodology rationalizations) ── independent
```

**Parallelization:** Tasks 1-6, 8-11, 13, 14, and 15 can all run in parallel. Tasks 7 and 12 are the only ones with dependencies.

---

## Execution Order

Recommended batch execution:

**Batch 1 (all independent work):**
- Tasks 1-6: Build `context/tdd-depth.md` (sequential appends to same file)
- Tasks 8-11: Build `context/debugging-techniques.md` (sequential appends to same file)
- Task 13: Spec reviewer suspicion framing
- Task 14: Human partner signals in debug mode
- Task 15: Methodology rationalization entries

**Batch 2 (wiring — depends on Batch 1):**
- Task 7: Wire tdd-depth.md into implementer agent
- Task 12: Wire debugging-techniques.md into debug mode

**Batch 3 (verification):**
- Run verification plan above
