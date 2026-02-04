# Bundle Health Check Feature Design

## Goal

Provide a CLI command that validates Amplifier bundles and catches common mistakes before runtime, with actionable error messages.

## Background

Currently, bundle validation happens at load/activation time, which means:
- **@mentions that don't resolve silently fail** - the mention just doesn't expand
- **Missing agent metadata logs a warning but continues** - agents become unusable in delegate tool
- **Module source issues only surface when activating** - late failure, poor error messages
- **YAML syntax errors produce cryptic exceptions** - no line numbers, no hints

Authors need fast feedback during development. A health check command would catch these issues before committing code or running sessions.

## Key Design Questions (Answered)

| Question | Decision | Rationale |
|----------|----------|-----------|
| Where should this live? | `amplifier-foundation` | Bundles are a foundation concept; existing `validator.py` is there |
| Primary interface? | CLI command | Works without starting a session; fast feedback loop |
| Validation modes? | Fast (local) + Comprehensive (network) | Fast for development, comprehensive for CI |
| Auto-fix capabilities? | Not in v1 | YAGNI - design for it, don't build it yet |
| Output format? | Human-readable + JSON flag | Console for humans, JSON for tooling/CI |

## Approaches Considered

### Approach A: Extend Existing BundleValidator

**Description**: Add new validation methods to the existing `validator.py` class.

**Pros**:
- Minimal new code
- Fits existing patterns
- Single validation entry point

**Cons**:
- Mixes fast and slow checks
- Harder to test individual checks
- Validator class grows unwieldy

### Approach B: New HealthChecker with Composable Checks ⭐ Recommended

**Description**: Create a separate `HealthChecker` class that orchestrates independent check functions. Each check returns a list of issues (errors/warnings).

**Pros**:
- Each check is independently testable
- Easy to add/remove checks
- Clear separation: fast vs comprehensive checks
- Can run subset of checks via CLI flags

**Cons**:
- More code than Approach A
- Two places for validation (existing validator + new checker)

### Approach C: Plugin-Based Architecture

**Description**: Health checks as separate modules that bundles can extend with custom checks.

**Pros**:
- Maximum flexibility
- Bundles can add domain-specific validation

**Cons**:
- Overkill for current needs
- Adds complexity for bundle authors
- YAGNI violation

**Decision**: **Approach B** - provides the right balance of testability, flexibility, and simplicity.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     CLI Layer                                 │
│  amplifier bundle check ./bundle.md [--comprehensive] [--json]│
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                   HealthChecker                               │
│  - run_checks(bundle_path, mode) → HealthReport              │
│  - registers checks by category (fast/comprehensive)          │
└────────────────────────┬─────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│ Fast Checks  │ │ Fast Checks  │ │  Slow Checks │
│ (Structure)  │ │  (Content)   │ │  (Network)   │
├──────────────┤ ├──────────────┤ ├──────────────┤
│ YAML syntax  │ │ @mention     │ │ Git source   │
│ Required     │ │  resolution  │ │  reachable   │
│  fields      │ │ Agent meta   │ │ Include URIs │
│ Schema       │ │ Local paths  │ │  fetch       │
└──────────────┘ └──────────────┘ └──────────────┘
```

---

## Components

### 1. HealthChecker Class

**Location**: `amplifier_foundation/health_checker.py`

```python
from dataclasses import dataclass, field
from enum import Enum
from pathlib import Path
from typing import Callable, Protocol

class Severity(Enum):
    ERROR = "error"      # Must fix - bundle won't work
    WARNING = "warning"  # Should fix - may cause issues
    INFO = "info"        # Suggestion - nice to have

@dataclass
class Issue:
    severity: Severity
    code: str            # e.g., "MISSING_META_NAME"
    message: str         # Human-readable description
    file_path: Path | None = None
    line: int | None = None
    fix_hint: str | None = None  # Actionable suggestion

@dataclass
class HealthReport:
    bundle_path: Path
    issues: list[Issue] = field(default_factory=list)
    checks_run: list[str] = field(default_factory=list)
    
    @property
    def has_errors(self) -> bool:
        return any(i.severity == Severity.ERROR for i in self.issues)
    
    @property  
    def passed(self) -> bool:
        return not self.has_errors

class CheckMode(Enum):
    FAST = "fast"                # Local checks only (~100ms)
    COMPREHENSIVE = "comprehensive"  # Include network checks (~seconds)

class HealthChecker:
    def __init__(self):
        self._fast_checks: list[CheckFunction] = []
        self._slow_checks: list[CheckFunction] = []
    
    def run(self, bundle_path: Path, mode: CheckMode = CheckMode.FAST) -> HealthReport:
        """Run health checks and return report."""
        ...
```

### 2. Check Functions

Each check is a simple function with signature:
```python
CheckFunction = Callable[[BundleData, Path], list[Issue]]
```

**Fast Checks** (always run):

| Check | Code | Validates |
|-------|------|-----------|
| `check_yaml_syntax` | `YAML_SYNTAX` | Frontmatter parses without error |
| `check_required_fields` | `MISSING_BUNDLE_NAME` | `bundle.name` exists |
| `check_module_format` | `INVALID_MODULE_FORMAT` | providers/tools/hooks are dicts with `module` |
| `check_agent_meta` | `MISSING_META_*` | Agent files have `meta.name` and `meta.description` |
| `check_mentions_resolve` | `UNRESOLVED_MENTION` | @mentions point to existing files |
| `check_local_paths` | `PATH_NOT_FOUND` | Local module sources exist |
| `check_namespace_consistency` | `UNKNOWN_NAMESPACE` | Includes reference known namespaces |

**Comprehensive Checks** (opt-in):

| Check | Code | Validates |
|-------|------|-----------|
| `check_git_sources_reachable` | `GIT_UNREACHABLE` | Git URLs can be fetched |
| `check_include_bundles_valid` | `INVALID_INCLUDE` | Included bundles load successfully |

### 3. CLI Integration

**Location**: `amplifier_foundation/cli/bundle_commands.py`

```bash
# Basic usage - fast checks only
amplifier bundle check ./bundle.md

# Comprehensive checks (includes network)
amplifier bundle check ./bundle.md --comprehensive

# JSON output for CI
amplifier bundle check ./bundle.md --json

# Check multiple bundles
amplifier bundle check ./bundle.md ./agents/*.md

# Recursive check (bundle + all agents)
amplifier bundle check ./bundle.md --recursive
```

**Exit codes**:
- `0` - All checks passed
- `1` - Errors found
- `2` - Invalid arguments

---

## Error Messages

Error messages follow this format:
```
[SEVERITY] CODE: message
  → file:line
  💡 fix_hint
```

**Examples**:

```
[ERROR] MISSING_META_NAME: Agent is missing required meta.name field
  → agents/my-agent.md:1
  💡 Add 'name: my-agent' under the meta: section

[ERROR] UNRESOLVED_MENTION: @foundation:nonexistent.md does not resolve to a file
  → bundle.md:45
  💡 Check the path exists or remove the @mention

[WARNING] MISSING_META_DESCRIPTION: Agent has no description - won't appear in delegate tool
  → agents/helper.md:1
  💡 Add 'description: |' under meta: with a clear explanation of when to use this agent

[ERROR] INVALID_MODULE_FORMAT: Tool entry must be a dict with 'module' key, got str
  → bundle.md:23
  💡 Change 'tool-name' to '- module: tool-name'
```

---

## Testing Strategy

### Unit Tests

```python
# tests/test_health_checker.py

def test_check_yaml_syntax_valid():
    """Valid YAML returns no issues."""
    issues = check_yaml_syntax(VALID_BUNDLE_DATA, VALID_PATH)
    assert issues == []

def test_check_yaml_syntax_invalid():
    """Invalid YAML returns YAML_SYNTAX error with line number."""
    issues = check_yaml_syntax_from_path(INVALID_YAML_PATH)
    assert len(issues) == 1
    assert issues[0].code == "YAML_SYNTAX"
    assert issues[0].line is not None

def test_check_agent_meta_missing_name():
    """Missing meta.name is an error."""
    issues = check_agent_meta(AGENT_WITHOUT_NAME, PATH)
    assert any(i.code == "MISSING_META_NAME" for i in issues)

def test_check_mentions_resolve_missing_file():
    """@mention to nonexistent file is an error."""
    issues = check_mentions_resolve(BUNDLE_WITH_BAD_MENTION, PATH)
    assert any(i.code == "UNRESOLVED_MENTION" for i in issues)
```

---

## Implementation Plan

### Phase 1: Core Infrastructure (~2 days)
- [ ] Create `HealthChecker` class and `Issue`/`HealthReport` dataclasses
- [ ] Implement check registration mechanism
- [ ] Add CLI command skeleton

### Phase 2: Fast Checks (~3 days)
- [ ] `check_yaml_syntax` - YAML parsing with error capture
- [ ] `check_required_fields` - bundle.name validation
- [ ] `check_module_format` - providers/tools/hooks structure
- [ ] `check_agent_meta` - meta.name and meta.description
- [ ] `check_mentions_resolve` - @mention file existence
- [ ] `check_local_paths` - local source path existence

### Phase 3: CLI Polish (~1 day)
- [ ] Human-readable output formatting
- [ ] JSON output format
- [ ] `--recursive` flag for checking agents
- [ ] Exit codes

### Phase 4: Comprehensive Checks (~2 days)
- [ ] `check_git_sources_reachable` - network check for git URLs
- [ ] `check_include_bundles_valid` - recursive include validation
- [ ] Timeout and error handling for network operations

### Phase 5: Documentation & Polish (~1 day)
- [ ] Update BUNDLE_GUIDE.md with health check docs
- [ ] Add error code reference documentation
- [ ] Integration tests for all error scenarios

---

## Summary

The bundle health check feature provides fast, actionable validation for Amplifier bundles. Key design decisions:

- **Composable checks** - Each validation is independent and testable
- **Two modes** - Fast (local-only) for development, comprehensive for CI
- **Actionable messages** - Every error includes a fix hint
- **CLI-first** - Works without starting a session, integrates with CI
