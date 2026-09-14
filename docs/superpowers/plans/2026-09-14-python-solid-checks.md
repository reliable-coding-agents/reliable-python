# Python SOLID Checks Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add Python-specific SOLID guidance, high-confidence local Liskov-substitution checks, and five focused maintainable-Python practices without making SOLID findings block the completion hook.

**Architecture:** Two on-demand skills separate SOLID principles from general Python practices. The existing dependency-free AST auditor gains focused SOLID and Python-practice passes; the Stop hook filters only the `SOLID` namespace from blocking selection.

**Tech Stack:** Python 3.10+ standard library (`ast`, `dataclasses`, `unittest`), Markdown skills, Claude/Codex plugin manifests.

**Spec:** `docs/superpowers/specs/2026-09-14-python-solid-checks-design.md`

## Global Constraints

- Analyze only syntax available in one parsed Python module; do not import or execute audited code.
- Automate only `SOLID03`; reserve `SOLID01`, `SOLID02`, `SOLID04`, and `SOLID05` for semantic review.
- Emit `SOLID03` as `warning`, but never select `SOLID*` findings to block the Stop hook.
- Preserve the existing stdlib-only runtime and Python 3.10 minimum.
- Keep functions at 60 code lines or fewer, block nesting at four levels or fewer, and loops mechanically bounded.
- Do not duplicate an overlapping `CS07` finding.
- Report only bare module-body calls as `PY001` and only annotation presence gaps on the existing public-function surface as `PY004`.

---

### Task 1: Detect incompatible local overrides

**Files:**
- Modify: `tests/test_audit_python.py`
- Modify: `skills/review-code-quality/scripts/audit_python.py`

**Interfaces:**
- Consumes: `ReviewContext.report`, the parsed `ReviewContext.tree`, and ordinary `ast.ClassDef`/function nodes.
- Produces: `_check_solid(context: ReviewContext) -> None`, called by `_run_checks`, and `SOLID03` findings anchored to the subclass override.

- [ ] **Step 1: Write failing behavior tests**

Add a `SolidPrincipleTests` class. Each expectation is derived from the Python call contract rather than helper output:

```python
class SolidPrincipleTests(unittest.TestCase):
    """Cover high-confidence Python Liskov-substitution findings."""

    def _solid_findings(self, source: str) -> list:
        return [
            item
            for item in AUDITOR.analyze_source(source, pathlib.Path("solid.py"))
            if item.code == "SOLID03"
        ]

    def test_reports_concrete_method_disabled_by_subclass(self) -> None:
        source = '''
class Writer:
    def write(self, payload):
        return payload

class ReadOnlyWriter(Writer):
    def write(self, payload):
        raise NotImplementedError
'''
        findings = self._solid_findings(source)
        self.assertEqual(len(findings), 1)
        self.assertIn("NotImplementedError", findings[0].message)

    def test_reports_sync_property_and_signature_incompatibilities(self) -> None:
        source = '''
class Service:
    def fetch(self, key, timeout=None):
        return key

    def dispatch(self, *items, **options):
        return items

    @property
    def status(self):
        return "ready"

class NarrowService(Service):
    async def fetch(self, key, timeout, region):
        return key

    def dispatch(self, item):
        return item

    def status(self):
        return "ready"
'''
        findings = self._solid_findings(source)
        self.assertEqual(len(findings), 3)
        message = "\n".join(item.message for item in findings)
        self.assertIn("synchronous", message)
        self.assertIn("property", message)
        self.assertIn("variadic", message)

    def test_accepts_compatible_and_abstract_overrides(self) -> None:
        source = '''
from abc import abstractmethod

class Service:
    @abstractmethod
    def required(self, key):
        raise NotImplementedError

    def fetch(self, key, timeout=None):
        return key

    def _internal(self):
        return None

class WideService(Service):
    def required(self, key):
        return key

    def fetch(self, key, timeout=None, *args, **kwargs):
        return key

    def _internal(self):
        raise NotImplementedError

class ExternalService(ExternalBase):
    def fetch(self, key):
        return key
'''
        self.assertEqual(self._solid_findings(source), [])
```

- [ ] **Step 2: Run tests and verify the red state**

Run:

```sh
python3 -m unittest tests.test_audit_python.SolidPrincipleTests -v
```

Expected: failures because no `SOLID03` finding exists yet.

- [ ] **Step 3: Implement the minimal AST pass**

Add small helpers named `_class_methods`, `_decorator_name`, `_method_kind`,
`_concrete_method`, `_raises_not_implemented`,
`_signature_incompatibilities`, `_override_incompatibilities`, and
`_check_solid`. Their typed inputs are respectively class nodes, decorator
expressions, function nodes, `ast.arguments`, and `ReviewContext`; collection
returns use concrete `dict[str, FunctionNode]` or `list[str]` annotations.

Use a `FunctionNode` type alias for the repeated union. Strip `self`/`cls` only for ordinary bound methods. Compare positional capacity and keyword names directly, respecting `*args` and `**kwargs`; report only narrowing facts. Ignore public methods with unsupported decorators except the directly visible `property`, `staticmethod`, `classmethod`, and `abstractmethod` markers. Resolve bases with a bounded projection over same-module `ast.Name` expressions.

Report once per override:

```python
context.report(
    code="SOLID03",
    severity="warning",
    node=override,
    message=(
        f"override {subclass.name}.{name} is not substitutable for "
        f"{base_class.name}.{name}: {'; '.join(reasons)}"
    ),
    remedy=(
        "Preserve the inherited call contract, or replace inheritance with "
        "composition or a narrower protocol."
    ),
)
```

Call `_check_solid(context)` from `_run_checks` after existing class checks.

- [ ] **Step 4: Run focused tests and refactor while green**

Run:

```sh
python3 -m unittest tests.test_audit_python.SolidPrincipleTests -v
python3 -m unittest tests.test_audit_python.AuditPythonTests -v
```

Expected: all focused tests pass. Split helpers if any function exceeds the repository’s size or nesting limits.

- [ ] **Step 5: Commit the checker slice**

```sh
git add tests/test_audit_python.py skills/review-code-quality/scripts/audit_python.py
git commit -m "feat: detect incompatible Python overrides"
```

### Task 2: Keep SOLID advisory in the Stop hook

**Files:**
- Modify: `tests/test_audit_python.py`
- Modify: `hooks/stop_quality_gate.py`

**Interfaces:**
- Consumes: auditor JSON findings and `RELIABLE_PYTHON_GATE`.
- Produces: `_selected_findings` that excludes codes beginning with `SOLID` before applying the existing severity policy.

- [ ] **Step 1: Write a failing hook integration test**

Create a temporary Git repository containing compatible baseline classes,
commit it, then change the subclass to disable a concrete method. Run the real
Stop hook and assert that it returns `{}` even though the direct auditor JSON
contains `SOLID03`. Add a second ordinary `eval` change and assert that `POT08`
still blocks.

```python
def test_stop_hook_treats_solid_findings_as_advisory(self) -> None:
    # Arrange a real changed-line Git diff and run the real hook subprocess.
    self.assertEqual(advisory_payload, {})
    self.assertEqual(blocking_payload["decision"], "block")
    self.assertIn("POT08", blocking_payload["reason"])
    self.assertNotIn("SOLID03", blocking_payload["reason"])
```

- [ ] **Step 2: Run the hook test and verify the red state**

Run:

```sh
python3 -m unittest tests.test_audit_python.HookIntegrationTests.test_stop_hook_treats_solid_findings_as_advisory -v
```

Expected: failure because the Stop hook currently selects every warning at gate level `all`.

- [ ] **Step 3: Filter the advisory namespace**

Add a named prefix constant and filter before severity selection:

```python
ADVISORY_CODE_PREFIXES = ("SOLID",)

def _selected_findings(payload: dict[str, Any], gate_level: str) -> list[dict[str, Any]]:
    raw = payload.get("findings", [])
    findings = [item for item in raw if isinstance(item, dict)]
    blocking = [
        item
        for item in findings
        if not str(item.get("code", "")).startswith(ADVISORY_CODE_PREFIXES)
    ]
    if gate_level == "errors":
        return [item for item in blocking if item.get("severity") == "error"]
    return blocking
```

- [ ] **Step 4: Run all hook integration tests**

Run:

```sh
python3 -m unittest tests.test_audit_python.HookIntegrationTests -v
```

Expected: all hook tests pass and existing gate behavior is unchanged.

- [ ] **Step 5: Commit the hook slice**

```sh
git add tests/test_audit_python.py hooks/stop_quality_gate.py
git commit -m "feat: keep SOLID findings advisory"
```

### Task 3: Check import safety and public annotations

**Files:**
- Modify: `tests/test_audit_python.py`
- Modify: `skills/review-code-quality/scripts/audit_python.py`

**Interfaces:**
- Consumes: module-body statements, collected `FunctionInfo` records, and the existing `_requires_docstring` public-surface predicate.
- Produces: `PY001` warnings for bare module-body calls and `PY004` warnings listing missing public-interface annotations.

- [ ] **Step 1: Write failing Python-practice tests**

Add tests using literal sources and exact finding projections:

```python
class PythonPracticeTests(unittest.TestCase):
    """Cover high-confidence maintainable-Python practices."""

    def test_reports_bare_module_call_but_accepts_main_guard(self) -> None:
        unsafe = "def connect():\n    return None\n\nconnect()\n"
        safe = (
            "def main() -> None:\n    connect()\n\n"
            "if __name__ == '__main__':\n    main()\n"
        )
        self.assertIn("PY001", codes(unsafe))
        self.assertNotIn("PY001", codes(safe))

    def test_reports_missing_public_interface_annotations(self) -> None:
        source = '''
def transform(value: str, limit=10):
    return value[:limit]

class Formatter:
    def render(self, value: str):
        return value
'''
        findings = AUDITOR.analyze_source(source, pathlib.Path("typed.py"))
        messages = [item.message for item in findings if item.code == "PY004"]
        self.assertEqual(len(messages), 2)
        self.assertTrue(any("limit" in message and "return" in message for message in messages))

    def test_accepts_complete_annotations_and_exempt_surfaces(self) -> None:
        source = '''
from typing import overload

def transform(value: str, limit: int = 10) -> str:
    return value[:limit]

def _helper(value):
    return value

@overload
def parse(value): ...

class Formatter:
    def render(self, value: str) -> str:
        return value
'''
        self.assertNotIn("PY004", codes(source))
```

- [ ] **Step 2: Run tests and verify the red state**

Run:

```sh
python3 -m unittest tests.test_audit_python.PythonPracticeTests -v
```

Expected: failures because `PY001` and `PY004` are not implemented.

- [ ] **Step 3: Add the two evidence-only checks**

Add `_check_import_safety(context: ReviewContext) -> None` and
`_check_public_annotations(context: ReviewContext, functions: Sequence[FunctionInfo]) -> None`.
For `PY001`, inspect only `context.tree.body` and report `ast.Expr` nodes whose
value is `ast.Call`. For `PY004`, reuse `_requires_docstring` as the public
surface, omit a leading `self`/`cls`, include positional-only, ordinary,
keyword-only, variadic, and keyword-variadic parameters, then report one
message listing all missing names plus `return` when `node.returns is None`.

```python
context.report(
    code="PY001",
    severity="warning",
    node=node,
    message="module executes a call during import",
    remedy="Move executable workflow into main() and call it under an __main__ guard.",
)
```

Call both checks from `_run_checks`. Keep `PY001` and `PY004` in the normal
blocking set; only the `SOLID` namespace is advisory.

- [ ] **Step 4: Run focused tests and the explicit self-audit**

Run:

```sh
python3 -m unittest tests.test_audit_python.PythonPracticeTests -v
python3 skills/review-code-quality/scripts/audit_python.py hooks tests skills
```

Expected: tests pass. If the auditor’s intentional module-loader call is
reported, add one adjacent `PY001` suppression with the concrete import reason.

- [ ] **Step 5: Commit the Python-practice checker slice**

```sh
git add tests/test_audit_python.py skills/review-code-quality/scripts/audit_python.py
git commit -m "feat: check Python import safety and annotations"
```

### Task 4: Ship the SOLID and Python-practices skills

**Files:**
- Create: `skills/applying-solid-principles/SKILL.md`
- Create: `skills/applying-solid-principles/agents/openai.yaml`
- Create: `skills/writing-maintainable-python/SKILL.md`
- Create: `skills/writing-maintainable-python/agents/openai.yaml`
- Modify: `skills/using-power-of-ten/SKILL.md`
- Modify: `skills/review-code-quality/SKILL.md`
- Modify: `README.md`
- Modify: `CLAUDE.md`
- Modify: `.claude-plugin/plugin.json`
- Modify: `.codex-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`
- Modify: `tests/test_audit_python.py`

**Interfaces:**
- Consumes: the rule IDs and checker limits from Tasks 1–2.
- Produces: discoverable cross-host SOLID and maintainable-Python guidance, plus consistent plugin version `0.4.0`.

- [ ] **Step 1: Establish the packaging red state**

Change `PackageConsistencyTests` to expect `0.4.0`, then run:

```sh
python3 -m unittest tests.test_audit_python.PackageConsistencyTests.test_host_manifests_and_marketplace_versions_match -v
```

Expected: failure showing the three manifests still contain `0.3.1`.

- [ ] **Step 2: Write the minimal on-demand skill**

Create a concise skill whose description begins with a discriminating trigger:

```yaml
---
name: applying-solid-principles
description: Use when designing or reviewing Python class hierarchies, protocols, interfaces, service boundaries, or dependency direction for SOLID-related maintainability risks.
---
```

Its body must:

- define `SOLID01` through `SOLID05` in Python terms;
- separate evidence from judgment and forbid threshold-only findings;
- explain the supported same-module `SOLID03` checks and advisory Stop policy;
- provide one compact compatible/incompatible override example;
- include a quick-reference table and common false-positive cautions;
- route detailed mechanical review to `review-code-quality` without claiming proof.

Create matching UI metadata:

```yaml
interface:
  display_name: "Apply Python SOLID"
  short_description: "Review Python boundaries with SOLID principles"
  default_prompt: "Use $applying-solid-principles to review this Python design for concrete SOLID risks."
```

- [ ] **Step 3: Route and document the feature**

Create `writing-maintainable-python` with matching `agents/openai.yaml`. Its
description triggers on Python entry points, import side effects, missing type
annotations, oversized/multi-purpose functions, and readability choices
between loops and comprehensions. Its body assigns `PY001`–`PY005`, gives the
guard-plus-`main()` pattern, points `PY003` to `POT04` and `SOLID01`, recommends
a configured type checker, and limits comprehensions to pure one-pass map or
filter expressions.

Use this frontmatter and UI metadata:

```yaml
---
name: writing-maintainable-python
description: Use when writing or reviewing Python entry points, import-time execution, public type annotations, oversized or multi-purpose functions, or loop-versus-comprehension readability.
---
```

```yaml
interface:
  display_name: "Write Maintainable Python"
  short_description: "Use explicit, readable Python program structure"
  default_prompt: "Use $writing-maintainable-python to improve this Python code's structure and explicitness."
```

Add compact load triggers to `using-power-of-ten`, add SOLID and the five
Python practices to semantic review, and document both skills, namespaces,
automated boundaries, and non-blocking SOLID Stop behavior in README/CLAUDE.md.
Change every required version occurrence and the README badge from `0.3.1` to
`0.4.0`.

- [ ] **Step 4: Validate the skill and packaging**

Run:

```sh
python3 /home/jihohan/.codex/skills/.system/skill-creator/scripts/quick_validate.py skills/applying-solid-principles
python3 /home/jihohan/.codex/skills/.system/skill-creator/scripts/quick_validate.py skills/writing-maintainable-python
python3 -m unittest tests.test_audit_python.PackageConsistencyTests -v
claude plugin validate .
```

Expected: skill validation, package tests, and manifest validation all pass.
Fresh-agent pressure testing is omitted because this session does not authorize
subagent delegation; perform a manual trigger and scope review instead.

- [ ] **Step 5: Commit the documentation and package slice**

```sh
git add .claude-plugin .codex-plugin CLAUDE.md README.md \
  skills/applying-solid-principles skills/writing-maintainable-python \
  skills/using-power-of-ten/SKILL.md \
  skills/review-code-quality/SKILL.md tests/test_audit_python.py
git commit -m "feat: add Python design guidance"
```

### Task 5: Full verification and semantic review

**Files:**
- Review only: all changed files

**Interfaces:**
- Consumes: completed implementation and documentation.
- Produces: fresh evidence that the feature and repository satisfy their contracts.

- [ ] **Step 1: Run the complete verification suite**

```sh
python3 -m unittest discover -s tests -v
python3 skills/review-code-quality/scripts/audit_python.py hooks tests skills
claude plugin validate .
git diff --check develop...HEAD
```

Expected: zero test failures, zero explicit audit findings, valid manifest,
and no whitespace errors.

- [ ] **Step 2: Review the complete diff against the spec**

Confirm every supported incompatibility has a failing-then-passing test,
compatible cases stay quiet, SOLID never blocks the Stop hook, existing gate
findings still block, both hosts receive the skill metadata, and all version
locations agree.

- [ ] **Step 3: Record the final branch state**

```sh
git status --short --branch
git log --oneline --decorate develop..HEAD
```

Expected: a clean `feat/python-solid-checks` worktree with the design,
checker, gate, guidance, and documentation commits present.
