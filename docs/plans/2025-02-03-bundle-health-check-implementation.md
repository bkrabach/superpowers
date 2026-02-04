# Bundle Health Check Implementation Plan (Phase 1 & 2)

> **Execution:** Use the subagent-driven-development workflow to implement this plan.

**Goal:** Implement core health check infrastructure and first 3 validation checks for Amplifier bundles.

**Architecture:** A `HealthChecker` class orchestrates independent check functions. Each check receives parsed bundle data and returns a list of `Issue` objects. The CLI (future) will call `HealthChecker.run()` and format the resulting `HealthReport`.

**Tech Stack:** Python 3.11+, pytest, dataclasses, pyyaml (already installed)

**Design Doc:** `docs/plans/2026-02-03-bundle-health-check-design.md`

---

## Phase 1: Core Infrastructure

### Task 1: Create Issue and Severity dataclasses

**Files:**
- Create: `amplifier-foundation/amplifier_foundation/health_checker.py`
- Test: `amplifier-foundation/tests/test_health_checker.py`

**Step 1: Write the failing test**

Create `amplifier-foundation/tests/test_health_checker.py`:

```python
"""Tests for bundle health checker."""

from __future__ import annotations

from pathlib import Path

import pytest

from amplifier_foundation.health_checker import Issue
from amplifier_foundation.health_checker import Severity


class TestSeverity:
    """Tests for Severity enum."""

    def test_severity_values(self) -> None:
        """Severity enum has expected values."""
        assert Severity.ERROR.value == "error"
        assert Severity.WARNING.value == "warning"
        assert Severity.INFO.value == "info"


class TestIssue:
    """Tests for Issue dataclass."""

    def test_create_minimal_issue(self) -> None:
        """Issue can be created with required fields only."""
        issue = Issue(
            severity=Severity.ERROR,
            code="TEST_CODE",
            message="Test message",
        )
        assert issue.severity == Severity.ERROR
        assert issue.code == "TEST_CODE"
        assert issue.message == "Test message"
        assert issue.file_path is None
        assert issue.line is None
        assert issue.fix_hint is None

    def test_create_full_issue(self) -> None:
        """Issue can be created with all fields."""
        issue = Issue(
            severity=Severity.WARNING,
            code="MISSING_FIELD",
            message="Field X is missing",
            file_path=Path("/test/bundle.md"),
            line=42,
            fix_hint="Add field X to the config",
        )
        assert issue.file_path == Path("/test/bundle.md")
        assert issue.line == 42
        assert issue.fix_hint == "Add field X to the config"
```

**Step 2: Run test to verify it fails**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py -v`

Expected: FAIL with `ModuleNotFoundError: No module named 'amplifier_foundation.health_checker'`

**Step 3: Write minimal implementation**

Create `amplifier-foundation/amplifier_foundation/health_checker.py`:

```python
"""Bundle health checker - validates bundles and catches common mistakes."""

from __future__ import annotations

from dataclasses import dataclass
from enum import Enum
from pathlib import Path


class Severity(Enum):
    """Severity level for health check issues."""

    ERROR = "error"  # Must fix - bundle won't work
    WARNING = "warning"  # Should fix - may cause issues
    INFO = "info"  # Suggestion - nice to have


@dataclass
class Issue:
    """A single health check issue.

    Attributes:
        severity: How serious the issue is.
        code: Machine-readable code (e.g., "MISSING_META_NAME").
        message: Human-readable description.
        file_path: Path to the file with the issue.
        line: Line number where issue occurs.
        fix_hint: Actionable suggestion for fixing.
    """

    severity: Severity
    code: str
    message: str
    file_path: Path | None = None
    line: int | None = None
    fix_hint: str | None = None
```

**Step 4: Run test to verify it passes**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestSeverity -v && uv run pytest tests/test_health_checker.py::TestIssue -v`

Expected: PASS

**Step 5: Commit**

```bash
cd amplifier-foundation
git add amplifier_foundation/health_checker.py tests/test_health_checker.py
git commit -m "feat(health-check): add Issue and Severity dataclasses"
```

---

### Task 2: Create HealthReport dataclass

**Files:**
- Modify: `amplifier-foundation/amplifier_foundation/health_checker.py`
- Modify: `amplifier-foundation/tests/test_health_checker.py`

**Step 1: Write the failing test**

Add to `amplifier-foundation/tests/test_health_checker.py`:

```python
from amplifier_foundation.health_checker import HealthReport


class TestHealthReport:
    """Tests for HealthReport dataclass."""

    def test_empty_report_passes(self) -> None:
        """Empty report (no issues) passes."""
        report = HealthReport(bundle_path=Path("/test/bundle.md"))
        assert report.passed is True
        assert report.has_errors is False
        assert report.issues == []
        assert report.checks_run == []

    def test_report_with_warning_passes(self) -> None:
        """Report with only warnings still passes."""
        report = HealthReport(
            bundle_path=Path("/test/bundle.md"),
            issues=[
                Issue(
                    severity=Severity.WARNING,
                    code="TEST_WARNING",
                    message="Just a warning",
                )
            ],
        )
        assert report.passed is True
        assert report.has_errors is False

    def test_report_with_error_fails(self) -> None:
        """Report with any error fails."""
        report = HealthReport(
            bundle_path=Path("/test/bundle.md"),
            issues=[
                Issue(
                    severity=Severity.ERROR,
                    code="TEST_ERROR",
                    message="This is an error",
                )
            ],
        )
        assert report.passed is False
        assert report.has_errors is True

    def test_report_tracks_checks_run(self) -> None:
        """Report tracks which checks were executed."""
        report = HealthReport(
            bundle_path=Path("/test/bundle.md"),
            checks_run=["check_yaml_syntax", "check_required_fields"],
        )
        assert len(report.checks_run) == 2
        assert "check_yaml_syntax" in report.checks_run
```

**Step 2: Run test to verify it fails**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestHealthReport -v`

Expected: FAIL with `ImportError: cannot import name 'HealthReport'`

**Step 3: Write minimal implementation**

Add to `amplifier-foundation/amplifier_foundation/health_checker.py` (after Issue class):

```python
from dataclasses import field


@dataclass
class HealthReport:
    """Result of running health checks on a bundle.

    Attributes:
        bundle_path: Path to the bundle that was checked.
        issues: List of issues found.
        checks_run: Names of checks that were executed.
    """

    bundle_path: Path
    issues: list[Issue] = field(default_factory=list)
    checks_run: list[str] = field(default_factory=list)

    @property
    def has_errors(self) -> bool:
        """Return True if any issues are errors."""
        return any(i.severity == Severity.ERROR for i in self.issues)

    @property
    def passed(self) -> bool:
        """Return True if no errors (warnings are OK)."""
        return not self.has_errors
```

Update the imports at the top of the file:

```python
from dataclasses import dataclass
from dataclasses import field
```

**Step 4: Run test to verify it passes**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestHealthReport -v`

Expected: PASS

**Step 5: Commit**

```bash
cd amplifier-foundation
git add amplifier_foundation/health_checker.py tests/test_health_checker.py
git commit -m "feat(health-check): add HealthReport dataclass"
```

---

### Task 3: Create CheckMode enum and HealthChecker class skeleton

**Files:**
- Modify: `amplifier-foundation/amplifier_foundation/health_checker.py`
- Modify: `amplifier-foundation/tests/test_health_checker.py`

**Step 1: Write the failing test**

Add to `amplifier-foundation/tests/test_health_checker.py`:

```python
from amplifier_foundation.health_checker import CheckMode
from amplifier_foundation.health_checker import HealthChecker


class TestCheckMode:
    """Tests for CheckMode enum."""

    def test_check_mode_values(self) -> None:
        """CheckMode enum has expected values."""
        assert CheckMode.FAST.value == "fast"
        assert CheckMode.COMPREHENSIVE.value == "comprehensive"


class TestHealthChecker:
    """Tests for HealthChecker class."""

    def test_create_health_checker(self) -> None:
        """HealthChecker can be instantiated."""
        checker = HealthChecker()
        assert checker is not None

    def test_run_returns_health_report(self, tmp_path: Path) -> None:
        """run() returns a HealthReport."""
        # Create a minimal valid bundle file
        bundle_file = tmp_path / "bundle.md"
        bundle_file.write_text(
            """---
bundle:
  name: test-bundle
---

# Test Bundle
"""
        )

        checker = HealthChecker()
        report = checker.run(bundle_file)

        assert isinstance(report, HealthReport)
        assert report.bundle_path == bundle_file

    def test_run_with_nonexistent_file(self, tmp_path: Path) -> None:
        """run() with nonexistent file returns error."""
        checker = HealthChecker()
        report = checker.run(tmp_path / "nonexistent.md")

        assert report.has_errors is True
        assert any(i.code == "FILE_NOT_FOUND" for i in report.issues)
```

**Step 2: Run test to verify it fails**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestCheckMode -v && uv run pytest tests/test_health_checker.py::TestHealthChecker -v`

Expected: FAIL with `ImportError: cannot import name 'CheckMode'`

**Step 3: Write minimal implementation**

Add to `amplifier-foundation/amplifier_foundation/health_checker.py`:

```python
from typing import Any
from typing import Callable


class CheckMode(Enum):
    """Mode for running health checks."""

    FAST = "fast"  # Local checks only (~100ms)
    COMPREHENSIVE = "comprehensive"  # Include network checks (~seconds)


# Type alias for check functions
# Check function signature: (frontmatter: dict, body: str, path: Path) -> list[Issue]
CheckFunction = Callable[[dict[str, Any], str, Path], list[Issue]]


class HealthChecker:
    """Orchestrates health checks on bundles.

    Runs composable check functions and aggregates results into a HealthReport.

    Usage:
        checker = HealthChecker()
        report = checker.run(Path("./bundle.md"))
        if not report.passed:
            for issue in report.issues:
                print(f"{issue.severity.value}: {issue.message}")
    """

    def __init__(self) -> None:
        """Initialize with empty check lists."""
        self._fast_checks: list[tuple[str, CheckFunction]] = []
        self._slow_checks: list[tuple[str, CheckFunction]] = []

    def register_fast_check(self, name: str, check: CheckFunction) -> None:
        """Register a fast (local-only) check."""
        self._fast_checks.append((name, check))

    def register_slow_check(self, name: str, check: CheckFunction) -> None:
        """Register a slow (network) check."""
        self._slow_checks.append((name, check))

    def run(
        self, bundle_path: Path, mode: CheckMode = CheckMode.FAST
    ) -> HealthReport:
        """Run health checks on a bundle.

        Args:
            bundle_path: Path to the bundle markdown file.
            mode: Which checks to run (FAST or COMPREHENSIVE).

        Returns:
            HealthReport with all issues found.
        """
        report = HealthReport(bundle_path=bundle_path)

        # Check file exists
        if not bundle_path.exists():
            report.issues.append(
                Issue(
                    severity=Severity.ERROR,
                    code="FILE_NOT_FOUND",
                    message=f"Bundle file not found: {bundle_path}",
                    file_path=bundle_path,
                    fix_hint="Check the path and ensure the file exists",
                )
            )
            return report

        # Read and parse file
        try:
            content = bundle_path.read_text()
            frontmatter, body = self._parse_frontmatter(content)
        except Exception as e:
            report.issues.append(
                Issue(
                    severity=Severity.ERROR,
                    code="FILE_READ_ERROR",
                    message=f"Failed to read bundle file: {e}",
                    file_path=bundle_path,
                )
            )
            return report

        # Run fast checks
        for name, check in self._fast_checks:
            report.checks_run.append(name)
            issues = check(frontmatter, body, bundle_path)
            report.issues.extend(issues)

        # Run slow checks if comprehensive mode
        if mode == CheckMode.COMPREHENSIVE:
            for name, check in self._slow_checks:
                report.checks_run.append(name)
                issues = check(frontmatter, body, bundle_path)
                report.issues.extend(issues)

        return report

    def _parse_frontmatter(self, content: str) -> tuple[dict[str, Any], str]:
        """Parse YAML frontmatter from markdown content."""
        from amplifier_foundation.io.frontmatter import parse_frontmatter

        return parse_frontmatter(content)
```

**Step 4: Run test to verify it passes**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestCheckMode -v && uv run pytest tests/test_health_checker.py::TestHealthChecker -v`

Expected: PASS

**Step 5: Commit**

```bash
cd amplifier-foundation
git add amplifier_foundation/health_checker.py tests/test_health_checker.py
git commit -m "feat(health-check): add CheckMode enum and HealthChecker class"
```

---

### Task 4: Add check registration and default checks initialization

**Files:**
- Modify: `amplifier-foundation/amplifier_foundation/health_checker.py`
- Modify: `amplifier-foundation/tests/test_health_checker.py`

**Step 1: Write the failing test**

Add to `amplifier-foundation/tests/test_health_checker.py`:

```python
from amplifier_foundation.health_checker import create_health_checker


class TestCheckRegistration:
    """Tests for check registration."""

    def test_register_fast_check(self) -> None:
        """Can register a fast check."""
        checker = HealthChecker()

        def dummy_check(
            frontmatter: dict, body: str, path: Path
        ) -> list[Issue]:
            return []

        checker.register_fast_check("dummy_check", dummy_check)
        assert len(checker._fast_checks) == 1

    def test_registered_check_is_called(self, tmp_path: Path) -> None:
        """Registered checks are called during run()."""
        checker = HealthChecker()
        call_count = {"count": 0}

        def counting_check(
            frontmatter: dict, body: str, path: Path
        ) -> list[Issue]:
            call_count["count"] += 1
            return []

        checker.register_fast_check("counting_check", counting_check)

        bundle_file = tmp_path / "bundle.md"
        bundle_file.write_text("---\nbundle:\n  name: test\n---\n")

        checker.run(bundle_file)
        assert call_count["count"] == 1

    def test_create_health_checker_with_defaults(self) -> None:
        """create_health_checker() returns checker with default checks."""
        checker = create_health_checker()
        assert len(checker._fast_checks) > 0


class TestChecksRunTracking:
    """Tests for tracking which checks were run."""

    def test_checks_run_is_populated(self, tmp_path: Path) -> None:
        """checks_run list contains names of executed checks."""
        checker = HealthChecker()

        def check_a(frontmatter: dict, body: str, path: Path) -> list[Issue]:
            return []

        def check_b(frontmatter: dict, body: str, path: Path) -> list[Issue]:
            return []

        checker.register_fast_check("check_a", check_a)
        checker.register_fast_check("check_b", check_b)

        bundle_file = tmp_path / "bundle.md"
        bundle_file.write_text("---\nbundle:\n  name: test\n---\n")

        report = checker.run(bundle_file)
        assert "check_a" in report.checks_run
        assert "check_b" in report.checks_run
```

**Step 2: Run test to verify it fails**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestCheckRegistration -v`

Expected: FAIL with `ImportError: cannot import name 'create_health_checker'`

**Step 3: Write minimal implementation**

Add to the end of `amplifier-foundation/amplifier_foundation/health_checker.py`:

```python
def create_health_checker() -> HealthChecker:
    """Create a HealthChecker with all default checks registered.

    Returns:
        HealthChecker configured with standard fast and slow checks.
    """
    checker = HealthChecker()

    # Fast checks will be registered here as they are implemented
    # Phase 2 will add: check_yaml_syntax, check_required_fields, check_module_format

    return checker
```

**Step 4: Run test to verify it passes**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestCheckRegistration -v && uv run pytest tests/test_health_checker.py::TestChecksRunTracking -v`

Expected: PASS

**Step 5: Commit**

```bash
cd amplifier-foundation
git add amplifier_foundation/health_checker.py tests/test_health_checker.py
git commit -m "feat(health-check): add create_health_checker factory and check registration"
```

---

### Task 5: Export health checker from package __init__.py

**Files:**
- Modify: `amplifier-foundation/amplifier_foundation/__init__.py`
- Modify: `amplifier-foundation/tests/test_health_checker.py`

**Step 1: Write the failing test**

Add to `amplifier-foundation/tests/test_health_checker.py`:

```python
class TestPublicAPI:
    """Tests for public API exports."""

    def test_can_import_from_package(self) -> None:
        """Key classes are importable from amplifier_foundation."""
        from amplifier_foundation import HealthChecker
        from amplifier_foundation import HealthReport
        from amplifier_foundation import Issue
        from amplifier_foundation import Severity
        from amplifier_foundation import create_health_checker

        assert HealthChecker is not None
        assert HealthReport is not None
        assert Issue is not None
        assert Severity is not None
        assert create_health_checker is not None
```

**Step 2: Run test to verify it fails**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestPublicAPI -v`

Expected: FAIL with `ImportError: cannot import name 'HealthChecker' from 'amplifier_foundation'`

**Step 3: Write minimal implementation**

Read the current `__init__.py` first, then add these exports. Add to `amplifier-foundation/amplifier_foundation/__init__.py`:

```python
from amplifier_foundation.health_checker import CheckMode
from amplifier_foundation.health_checker import HealthChecker
from amplifier_foundation.health_checker import HealthReport
from amplifier_foundation.health_checker import Issue
from amplifier_foundation.health_checker import Severity
from amplifier_foundation.health_checker import create_health_checker
```

And add to the `__all__` list:

```python
    "CheckMode",
    "HealthChecker",
    "HealthReport",
    "Issue",
    "Severity",
    "create_health_checker",
```

**Step 4: Run test to verify it passes**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestPublicAPI -v`

Expected: PASS

**Step 5: Commit**

```bash
cd amplifier-foundation
git add amplifier_foundation/__init__.py tests/test_health_checker.py
git commit -m "feat(health-check): export health checker types from package"
```

---

## Phase 2: First 3 Fast Checks

### Task 6: Implement check_yaml_syntax

**Files:**
- Modify: `amplifier-foundation/amplifier_foundation/health_checker.py`
- Modify: `amplifier-foundation/tests/test_health_checker.py`

**Step 1: Write the failing test**

Add to `amplifier-foundation/tests/test_health_checker.py`:

```python
from amplifier_foundation.health_checker import check_yaml_syntax


class TestCheckYamlSyntax:
    """Tests for check_yaml_syntax check function."""

    def test_valid_yaml_returns_no_issues(self, tmp_path: Path) -> None:
        """Valid YAML frontmatter returns empty list."""
        bundle_path = tmp_path / "bundle.md"
        frontmatter = {"bundle": {"name": "test"}}
        body = "# Test"

        issues = check_yaml_syntax(frontmatter, body, bundle_path)
        assert issues == []

    def test_detects_yaml_error_via_checker(self, tmp_path: Path) -> None:
        """HealthChecker catches YAML syntax errors before parsing."""
        bundle_file = tmp_path / "bundle.md"
        # Invalid YAML: tab character where spaces expected
        bundle_file.write_text(
            """---
bundle:
\tname: test
---
"""
        )

        checker = create_health_checker()
        report = checker.run(bundle_file)

        # Should have a YAML_SYNTAX error
        assert any(i.code == "YAML_SYNTAX" for i in report.issues)

    def test_yaml_error_includes_helpful_message(self, tmp_path: Path) -> None:
        """YAML error includes line info when available."""
        bundle_file = tmp_path / "bundle.md"
        # Invalid YAML: unclosed quote
        bundle_file.write_text(
            '''---
bundle:
  name: "unclosed
---
'''
        )

        checker = create_health_checker()
        report = checker.run(bundle_file)

        yaml_errors = [i for i in report.issues if i.code == "YAML_SYNTAX"]
        assert len(yaml_errors) == 1
        assert yaml_errors[0].fix_hint is not None
```

**Step 2: Run test to verify it fails**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestCheckYamlSyntax -v`

Expected: FAIL with `ImportError: cannot import name 'check_yaml_syntax'`

**Step 3: Write minimal implementation**

Add to `amplifier-foundation/amplifier_foundation/health_checker.py` (before `create_health_checker`):

```python
import yaml


def check_yaml_syntax(
    frontmatter: dict[str, Any], body: str, bundle_path: Path
) -> list[Issue]:
    """Check that YAML frontmatter has valid syntax.

    Note: This check runs AFTER parsing succeeds. For invalid YAML,
    the error is caught in HealthChecker.run() before this check runs.
    This check validates the parsed structure.

    Args:
        frontmatter: Parsed frontmatter dict.
        body: Markdown body text.
        bundle_path: Path to the bundle file.

    Returns:
        List of issues (empty if valid).
    """
    # If we got here, YAML parsed successfully
    # This check is mostly a placeholder that confirms parsing worked
    return []
```

Update the `HealthChecker.run()` method to catch YAML errors with proper code. Replace the try/except in `run()`:

```python
    def run(
        self, bundle_path: Path, mode: CheckMode = CheckMode.FAST
    ) -> HealthReport:
        """Run health checks on a bundle.

        Args:
            bundle_path: Path to the bundle markdown file.
            mode: Which checks to run (FAST or COMPREHENSIVE).

        Returns:
            HealthReport with all issues found.
        """
        report = HealthReport(bundle_path=bundle_path)

        # Check file exists
        if not bundle_path.exists():
            report.issues.append(
                Issue(
                    severity=Severity.ERROR,
                    code="FILE_NOT_FOUND",
                    message=f"Bundle file not found: {bundle_path}",
                    file_path=bundle_path,
                    fix_hint="Check the path and ensure the file exists",
                )
            )
            return report

        # Read and parse file
        try:
            content = bundle_path.read_text()
        except Exception as e:
            report.issues.append(
                Issue(
                    severity=Severity.ERROR,
                    code="FILE_READ_ERROR",
                    message=f"Failed to read bundle file: {e}",
                    file_path=bundle_path,
                )
            )
            return report

        # Parse frontmatter (catch YAML errors)
        try:
            frontmatter, body = self._parse_frontmatter(content)
        except yaml.YAMLError as e:
            # Extract line number if available
            line = None
            if hasattr(e, "problem_mark") and e.problem_mark:
                line = e.problem_mark.line + 1  # 1-indexed

            report.issues.append(
                Issue(
                    severity=Severity.ERROR,
                    code="YAML_SYNTAX",
                    message=f"Invalid YAML in frontmatter: {e}",
                    file_path=bundle_path,
                    line=line,
                    fix_hint="Check for common YAML issues: incorrect indentation, missing quotes, or tab characters",
                )
            )
            return report

        # Run fast checks
        for name, check in self._fast_checks:
            report.checks_run.append(name)
            issues = check(frontmatter, body, bundle_path)
            report.issues.extend(issues)

        # Run slow checks if comprehensive mode
        if mode == CheckMode.COMPREHENSIVE:
            for name, check in self._slow_checks:
                report.checks_run.append(name)
                issues = check(frontmatter, body, bundle_path)
                report.issues.extend(issues)

        return report
```

Update `create_health_checker()` to register the check:

```python
def create_health_checker() -> HealthChecker:
    """Create a HealthChecker with all default checks registered.

    Returns:
        HealthChecker configured with standard fast and slow checks.
    """
    checker = HealthChecker()

    # Fast checks - structure validation
    checker.register_fast_check("check_yaml_syntax", check_yaml_syntax)

    return checker
```

**Step 4: Run test to verify it passes**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestCheckYamlSyntax -v`

Expected: PASS

**Step 5: Commit**

```bash
cd amplifier-foundation
git add amplifier_foundation/health_checker.py tests/test_health_checker.py
git commit -m "feat(health-check): add check_yaml_syntax for YAML validation"
```

---

### Task 7: Implement check_required_fields

**Files:**
- Modify: `amplifier-foundation/amplifier_foundation/health_checker.py`
- Modify: `amplifier-foundation/tests/test_health_checker.py`

**Step 1: Write the failing test**

Add to `amplifier-foundation/tests/test_health_checker.py`:

```python
from amplifier_foundation.health_checker import check_required_fields


class TestCheckRequiredFields:
    """Tests for check_required_fields check function."""

    def test_valid_bundle_returns_no_issues(self, tmp_path: Path) -> None:
        """Bundle with name returns no issues."""
        frontmatter = {"bundle": {"name": "my-bundle"}}
        issues = check_required_fields(frontmatter, "", tmp_path / "bundle.md")
        assert issues == []

    def test_missing_bundle_section_is_error(self, tmp_path: Path) -> None:
        """Missing 'bundle' section is an error."""
        frontmatter = {"something": "else"}
        issues = check_required_fields(frontmatter, "", tmp_path / "bundle.md")

        assert len(issues) == 1
        assert issues[0].code == "MISSING_BUNDLE_SECTION"
        assert issues[0].severity == Severity.ERROR

    def test_missing_bundle_name_is_error(self, tmp_path: Path) -> None:
        """Missing bundle.name is an error."""
        frontmatter = {"bundle": {"version": "1.0"}}
        issues = check_required_fields(frontmatter, "", tmp_path / "bundle.md")

        assert len(issues) == 1
        assert issues[0].code == "MISSING_BUNDLE_NAME"
        assert issues[0].severity == Severity.ERROR

    def test_empty_bundle_name_is_error(self, tmp_path: Path) -> None:
        """Empty string for bundle.name is an error."""
        frontmatter = {"bundle": {"name": ""}}
        issues = check_required_fields(frontmatter, "", tmp_path / "bundle.md")

        assert len(issues) == 1
        assert issues[0].code == "MISSING_BUNDLE_NAME"

    def test_agent_file_without_bundle_section_ok(self, tmp_path: Path) -> None:
        """Agent files (with meta section) don't need bundle section."""
        frontmatter = {"meta": {"name": "my-agent", "description": "test"}}
        issues = check_required_fields(frontmatter, "", tmp_path / "agent.md")

        # Should pass - it's an agent file, not a bundle
        assert issues == []

    def test_error_includes_fix_hint(self, tmp_path: Path) -> None:
        """Errors include helpful fix hints."""
        frontmatter = {"bundle": {}}
        issues = check_required_fields(frontmatter, "", tmp_path / "bundle.md")

        assert issues[0].fix_hint is not None
        assert "name" in issues[0].fix_hint.lower()
```

**Step 2: Run test to verify it fails**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestCheckRequiredFields -v`

Expected: FAIL with `ImportError: cannot import name 'check_required_fields'`

**Step 3: Write minimal implementation**

Add to `amplifier-foundation/amplifier_foundation/health_checker.py` (after `check_yaml_syntax`):

```python
def check_required_fields(
    frontmatter: dict[str, Any], body: str, bundle_path: Path
) -> list[Issue]:
    """Check that required fields are present.

    For bundle files: bundle.name is required.
    For agent files: meta section indicates it's an agent (different rules).

    Args:
        frontmatter: Parsed frontmatter dict.
        body: Markdown body text.
        bundle_path: Path to the bundle file.

    Returns:
        List of issues found.
    """
    issues: list[Issue] = []

    # Check if this is an agent file (has meta section)
    if "meta" in frontmatter:
        # Agent files have different requirements - handled by check_agent_meta
        return issues

    # For bundle files, require bundle.name
    bundle_section = frontmatter.get("bundle")

    if bundle_section is None:
        issues.append(
            Issue(
                severity=Severity.ERROR,
                code="MISSING_BUNDLE_SECTION",
                message="Bundle file missing 'bundle:' section in frontmatter",
                file_path=bundle_path,
                fix_hint="Add 'bundle:\\n  name: your-bundle-name' to the frontmatter",
            )
        )
        return issues

    if not isinstance(bundle_section, dict):
        issues.append(
            Issue(
                severity=Severity.ERROR,
                code="INVALID_BUNDLE_SECTION",
                message="'bundle:' must be a mapping, not a scalar",
                file_path=bundle_path,
                fix_hint="Ensure 'bundle:' is followed by indented key-value pairs",
            )
        )
        return issues

    name = bundle_section.get("name")
    if not name:
        issues.append(
            Issue(
                severity=Severity.ERROR,
                code="MISSING_BUNDLE_NAME",
                message="Bundle is missing required 'name' field",
                file_path=bundle_path,
                fix_hint="Add 'name: your-bundle-name' under the 'bundle:' section",
            )
        )

    return issues
```

Update `create_health_checker()` to register the check:

```python
def create_health_checker() -> HealthChecker:
    """Create a HealthChecker with all default checks registered.

    Returns:
        HealthChecker configured with standard fast and slow checks.
    """
    checker = HealthChecker()

    # Fast checks - structure validation
    checker.register_fast_check("check_yaml_syntax", check_yaml_syntax)
    checker.register_fast_check("check_required_fields", check_required_fields)

    return checker
```

**Step 4: Run test to verify it passes**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestCheckRequiredFields -v`

Expected: PASS

**Step 5: Commit**

```bash
cd amplifier-foundation
git add amplifier_foundation/health_checker.py tests/test_health_checker.py
git commit -m "feat(health-check): add check_required_fields for bundle.name validation"
```

---

### Task 8: Implement check_module_format

**Files:**
- Modify: `amplifier-foundation/amplifier_foundation/health_checker.py`
- Modify: `amplifier-foundation/tests/test_health_checker.py`

**Step 1: Write the failing test**

Add to `amplifier-foundation/tests/test_health_checker.py`:

```python
from amplifier_foundation.health_checker import check_module_format


class TestCheckModuleFormat:
    """Tests for check_module_format check function."""

    def test_valid_modules_return_no_issues(self, tmp_path: Path) -> None:
        """Properly formatted module lists return no issues."""
        frontmatter = {
            "bundle": {"name": "test"},
            "providers": [{"module": "provider-anthropic"}],
            "tools": [{"module": "tool-bash", "config": {"timeout": 30}}],
            "hooks": [{"module": "hook-status"}],
        }
        issues = check_module_format(frontmatter, "", tmp_path / "bundle.md")
        assert issues == []

    def test_empty_lists_return_no_issues(self, tmp_path: Path) -> None:
        """Empty module lists are valid."""
        frontmatter = {
            "bundle": {"name": "test"},
            "providers": [],
            "tools": [],
        }
        issues = check_module_format(frontmatter, "", tmp_path / "bundle.md")
        assert issues == []

    def test_string_instead_of_dict_is_error(self, tmp_path: Path) -> None:
        """String entry instead of dict is an error."""
        frontmatter = {
            "bundle": {"name": "test"},
            "providers": ["provider-anthropic"],  # Should be dict
        }
        issues = check_module_format(frontmatter, "", tmp_path / "bundle.md")

        assert len(issues) == 1
        assert issues[0].code == "INVALID_MODULE_FORMAT"
        assert "providers[0]" in issues[0].message
        assert issues[0].fix_hint is not None

    def test_missing_module_key_is_error(self, tmp_path: Path) -> None:
        """Dict without 'module' key is an error."""
        frontmatter = {
            "bundle": {"name": "test"},
            "tools": [{"config": {"timeout": 30}}],  # Missing 'module'
        }
        issues = check_module_format(frontmatter, "", tmp_path / "bundle.md")

        assert len(issues) == 1
        assert issues[0].code == "MISSING_MODULE_KEY"
        assert "tools[0]" in issues[0].message

    def test_multiple_errors_reported(self, tmp_path: Path) -> None:
        """Multiple format errors are all reported."""
        frontmatter = {
            "bundle": {"name": "test"},
            "providers": ["bad-provider"],
            "tools": [{"config": {}}],
            "hooks": ["bad-hook"],
        }
        issues = check_module_format(frontmatter, "", tmp_path / "bundle.md")

        assert len(issues) == 3

    def test_non_list_module_section_is_error(self, tmp_path: Path) -> None:
        """Module section that's not a list is an error."""
        frontmatter = {
            "bundle": {"name": "test"},
            "providers": {"module": "provider-anthropic"},  # Should be list
        }
        issues = check_module_format(frontmatter, "", tmp_path / "bundle.md")

        assert len(issues) == 1
        assert issues[0].code == "INVALID_MODULE_LIST"

    def test_agent_files_skip_module_checks(self, tmp_path: Path) -> None:
        """Agent files (with meta) skip module format checks."""
        frontmatter = {
            "meta": {"name": "agent"},
            "tools": ["tool-name"],  # Would be error in bundle
        }
        issues = check_module_format(frontmatter, "", tmp_path / "agent.md")

        # Agent files have different tool format rules
        assert issues == []
```

**Step 2: Run test to verify it fails**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestCheckModuleFormat -v`

Expected: FAIL with `ImportError: cannot import name 'check_module_format'`

**Step 3: Write minimal implementation**

Add to `amplifier-foundation/amplifier_foundation/health_checker.py` (after `check_required_fields`):

```python
def check_module_format(
    frontmatter: dict[str, Any], body: str, bundle_path: Path
) -> list[Issue]:
    """Check that module lists (providers, tools, hooks) have correct format.

    Each entry must be a dict with a 'module' key.

    Args:
        frontmatter: Parsed frontmatter dict.
        body: Markdown body text.
        bundle_path: Path to the bundle file.

    Returns:
        List of issues found.
    """
    issues: list[Issue] = []

    # Skip for agent files
    if "meta" in frontmatter:
        return issues

    module_sections = ["providers", "tools", "hooks"]

    for section_name in module_sections:
        section = frontmatter.get(section_name)

        if section is None:
            continue

        # Section must be a list
        if not isinstance(section, list):
            issues.append(
                Issue(
                    severity=Severity.ERROR,
                    code="INVALID_MODULE_LIST",
                    message=f"'{section_name}' must be a list, got {type(section).__name__}",
                    file_path=bundle_path,
                    fix_hint=f"Change '{section_name}:' to a list with '- module: ...' entries",
                )
            )
            continue

        # Each entry must be a dict with 'module' key
        for idx, entry in enumerate(section):
            if not isinstance(entry, dict):
                issues.append(
                    Issue(
                        severity=Severity.ERROR,
                        code="INVALID_MODULE_FORMAT",
                        message=f"{section_name}[{idx}] must be a dict with 'module' key, got {type(entry).__name__}",
                        file_path=bundle_path,
                        fix_hint=f"Change '{entry}' to '- module: {entry}'",
                    )
                )
                continue

            if "module" not in entry:
                issues.append(
                    Issue(
                        severity=Severity.ERROR,
                        code="MISSING_MODULE_KEY",
                        message=f"{section_name}[{idx}] is missing required 'module' key",
                        file_path=bundle_path,
                        fix_hint="Add 'module: module-name' to the entry",
                    )
                )

    return issues
```

Update `create_health_checker()` to register the check:

```python
def create_health_checker() -> HealthChecker:
    """Create a HealthChecker with all default checks registered.

    Returns:
        HealthChecker configured with standard fast and slow checks.
    """
    checker = HealthChecker()

    # Fast checks - structure validation
    checker.register_fast_check("check_yaml_syntax", check_yaml_syntax)
    checker.register_fast_check("check_required_fields", check_required_fields)
    checker.register_fast_check("check_module_format", check_module_format)

    return checker
```

**Step 4: Run test to verify it passes**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestCheckModuleFormat -v`

Expected: PASS

**Step 5: Commit**

```bash
cd amplifier-foundation
git add amplifier_foundation/health_checker.py tests/test_health_checker.py
git commit -m "feat(health-check): add check_module_format for providers/tools/hooks validation"
```

---

### Task 9: Add integration test and export check functions

**Files:**
- Modify: `amplifier-foundation/amplifier_foundation/health_checker.py`
- Modify: `amplifier-foundation/tests/test_health_checker.py`

**Step 1: Write the failing test**

Add to `amplifier-foundation/tests/test_health_checker.py`:

```python
class TestIntegration:
    """Integration tests for the complete health checker."""

    def test_valid_bundle_passes_all_checks(self, tmp_path: Path) -> None:
        """A well-formed bundle passes all checks."""
        bundle_file = tmp_path / "bundle.md"
        bundle_file.write_text(
            """---
bundle:
  name: test-bundle
  version: "1.0.0"

providers:
  - module: provider-anthropic
    config:
      api_key_env: ANTHROPIC_API_KEY

tools:
  - module: tool-bash

hooks:
  - module: hook-status
---

# Test Bundle

This is a test bundle.
"""
        )

        checker = create_health_checker()
        report = checker.run(bundle_file)

        assert report.passed is True
        assert len(report.checks_run) >= 3
        assert report.issues == []

    def test_multiple_issues_reported(self, tmp_path: Path) -> None:
        """Multiple issues in one bundle are all reported."""
        bundle_file = tmp_path / "bundle.md"
        bundle_file.write_text(
            """---
bundle:
  version: "1.0"

providers:
  - provider-anthropic
  - module: provider-openai

tools:
  - config: {}
---
"""
        )

        checker = create_health_checker()
        report = checker.run(bundle_file)

        assert report.passed is False
        # Should have: MISSING_BUNDLE_NAME, INVALID_MODULE_FORMAT (providers[0]),
        # MISSING_MODULE_KEY (tools[0])
        assert len(report.issues) >= 3

        codes = [i.code for i in report.issues]
        assert "MISSING_BUNDLE_NAME" in codes
        assert "INVALID_MODULE_FORMAT" in codes
        assert "MISSING_MODULE_KEY" in codes

    def test_report_summary(self, tmp_path: Path) -> None:
        """Report provides useful summary information."""
        bundle_file = tmp_path / "bundle.md"
        bundle_file.write_text(
            """---
bundle:
  name: test
providers:
  - bad-entry
---
"""
        )

        checker = create_health_checker()
        report = checker.run(bundle_file)

        # Can count errors vs warnings
        errors = [i for i in report.issues if i.severity == Severity.ERROR]
        warnings = [i for i in report.issues if i.severity == Severity.WARNING]

        assert len(errors) >= 1
        assert isinstance(report.checks_run, list)


class TestCheckFunctionExports:
    """Test that check functions are exported."""

    def test_check_functions_importable(self) -> None:
        """Check functions can be imported directly."""
        from amplifier_foundation.health_checker import check_module_format
        from amplifier_foundation.health_checker import check_required_fields
        from amplifier_foundation.health_checker import check_yaml_syntax

        assert callable(check_yaml_syntax)
        assert callable(check_required_fields)
        assert callable(check_module_format)
```

**Step 2: Run test to verify it passes**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py::TestIntegration -v && uv run pytest tests/test_health_checker.py::TestCheckFunctionExports -v`

Expected: PASS (these tests should pass with existing code)

**Step 3: Run full test suite**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py -v`

Expected: All tests PASS

**Step 4: Commit**

```bash
cd amplifier-foundation
git add tests/test_health_checker.py
git commit -m "test(health-check): add integration tests for Phase 1 & 2"
```

---

### Task 10: Run full test suite and create fixtures directory

**Files:**
- Create: `amplifier-foundation/test-fixtures/health-check/valid-bundle.md`
- Create: `amplifier-foundation/test-fixtures/health-check/invalid-yaml.md`
- Create: `amplifier-foundation/test-fixtures/health-check/missing-name.md`

**Step 1: Create test fixtures**

Create `amplifier-foundation/test-fixtures/health-check/valid-bundle.md`:

```markdown
---
bundle:
  name: valid-test-bundle
  version: "1.0.0"
  description: A valid bundle for testing

providers:
  - module: provider-anthropic
    config:
      api_key_env: ANTHROPIC_API_KEY

tools:
  - module: tool-bash
  - module: tool-read-file

hooks:
  - module: hook-status
---

# Valid Test Bundle

This bundle is used for health checker tests.
```

Create `amplifier-foundation/test-fixtures/health-check/invalid-yaml.md`:

```markdown
---
bundle:
	name: bad-indent
  version: "1.0"
---
```

Create `amplifier-foundation/test-fixtures/health-check/missing-name.md`:

```markdown
---
bundle:
  version: "1.0.0"
  description: Missing name field

providers:
  - module: provider-anthropic
---

# Missing Name Bundle
```

**Step 2: Run complete test suite**

Run: `cd amplifier-foundation && uv run pytest tests/test_health_checker.py -v --tb=short`

Expected: All tests PASS

**Step 3: Run python_check for code quality**

Run: `cd amplifier-foundation && uv run ruff check amplifier_foundation/health_checker.py && uv run ruff format --check amplifier_foundation/health_checker.py`

Fix any issues if needed.

**Step 4: Commit fixtures and any fixes**

```bash
cd amplifier-foundation
git add test-fixtures/health-check/
git commit -m "test(health-check): add test fixtures for health checker"
```

---

## Summary

After completing all tasks, you will have:

### New Files
- `amplifier_foundation/health_checker.py` - Core health checker implementation
- `tests/test_health_checker.py` - Comprehensive test suite
- `test-fixtures/health-check/*.md` - Test fixture files

### Modified Files
- `amplifier_foundation/__init__.py` - Exports for public API

### Implemented Features
1. **Severity enum** - ERROR, WARNING, INFO levels
2. **Issue dataclass** - Captures validation issues with file/line/hint
3. **HealthReport dataclass** - Aggregates issues with passed/has_errors properties
4. **CheckMode enum** - FAST vs COMPREHENSIVE modes
5. **HealthChecker class** - Orchestrates checks, handles file reading and YAML parsing
6. **check_yaml_syntax** - Validates YAML frontmatter syntax
7. **check_required_fields** - Validates bundle.name is present
8. **check_module_format** - Validates providers/tools/hooks list structure
9. **create_health_checker()** - Factory function with default checks registered

### Test Coverage
- Unit tests for all dataclasses and enums
- Unit tests for each check function
- Integration tests for complete workflows
- Edge case handling (missing files, invalid YAML, etc.)

---

Plan complete and saved to `docs/plans/2025-02-03-bundle-health-check-implementation.md`.

**Execution options:**

1. **Subagent-Driven (this session)** 
   - Fresh agent per task
   - Two-stage review (spec then quality)
   - Fast iteration

2. **Parallel Session**
   - Open new session for execution
   - Batch execution with human checkpoints

Which approach?
