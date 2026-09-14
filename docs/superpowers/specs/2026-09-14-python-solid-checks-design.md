# Python SOLID Checks Design

## Objective

Add Python-specific SOLID guidance and a deliberately small set of
high-confidence static checks. The plugin should help reviewers notice concrete
substitution failures without claiming that an AST can prove architectural
quality or forcing speculative heuristics into the completion gate.

## Scope

The feature covers all five principles as review guidance:

| ID | Principle | Static status in this release |
|---|---|---|
| `SOLID01` | Single Responsibility Principle | Semantic review only |
| `SOLID02` | Open/Closed Principle | Semantic review only |
| `SOLID03` | Liskov Substitution Principle | High-confidence local checks |
| `SOLID04` | Interface Segregation Principle | Semantic review only |
| `SOLID05` | Dependency Inversion Principle | Semantic review only |

The checker analyzes only Python syntax in one module at a time. It does not
perform imports, execute code, resolve external bases, infer types, or assign
thresholds to responsibilities, interfaces, or dependencies.

## Skill architecture

Create an on-demand `applying-solid-principles` skill with the Codex metadata
required by this plugin. The skill explains the five principles in Python,
distinguishes evidence from architectural judgment, and directs reviewers to
the mechanical `SOLID03` checks.

Keep the always-injected `using-power-of-ten` skill compact. Add one routing
sentence telling agents to load the SOLID skill when changing Python class
hierarchies, protocols, or dependency boundaries. Extend `review-code-quality`
so its semantic review explicitly covers the five principles and reads the new
reference.

## Static-analysis boundary

Add a dedicated `_check_solid` pass to `audit_python.py`. It resolves a base
only when the base expression is a simple name bound to another class in the
same parsed module. It compares public methods declared directly on that base
and subclass. Private/protected names and dunder methods are outside the public
substitution contract and are skipped.

One `SOLID03` warning is emitted for an overriding method when any of these
facts is visible:

1. A concrete base method is replaced by a body that unconditionally raises
   `NotImplementedError`.
2. The override changes a normal method into a property or a property into a
   normal method.
3. The override changes a synchronous method into an asynchronous method or
   vice versa.
4. The override narrows calls accepted by the base signature by removing an
   accepted positional or keyword form, removing variadic acceptance, making
   an optional argument required, or adding a new required argument.

The implementation may combine multiple proven incompatibilities into one
finding for the override. The finding anchors to the overriding method so
changed-line filtering behaves normally. Its remedy is to preserve the base
call contract or replace inheritance with composition/a narrower protocol.

Abstract base methods and stub bodies are not treated as concrete inherited
behavior. Static, class, custom-descriptor, decorated-signature, generated, and
external inheritance cases are not inferred beyond facts directly represented
by the supported syntax. No `CS07` finding is emitted for the same condition;
`SOLID03` is the more precise identifier and avoids duplicate advice.

## Gate behavior

`SOLID03` findings use `warning` severity and appear in direct text and JSON
audits. The Stop hook excludes every `SOLID*` finding from its blocking
selection, regardless of `RELIABLE_PYTHON_GATE`. Existing Power of Ten,
code-smell, documentation, and infrastructure findings retain their current
behavior.

This makes SOLID evidence available to explicit audits without turning a
principle-oriented checker into an automatic architectural veto. Suppressions
remain available for direct audits using the existing adjacent
`# quality: ignore[SOLID03] - rationale` syntax.

## Additional Python practices

Ship a second on-demand `writing-maintainable-python` skill so general Python
advice is not mislabeled as SOLID. It assigns stable identifiers to the five
requested practices:

| ID | Practice | Static status in this release |
|---|---|---|
| `PY001` | Guard executable module-level code | Bare top-level call expressions |
| `PY002` | Assemble executable workflows in `main()` | Guidance only |
| `PY003` | Keep functions small and single-purpose | Existing `POT04`; semantic `SOLID01` review |
| `PY004` | Annotate public function interfaces | Missing parameter or return annotations |
| `PY005` | Prefer simple comprehensions for clear projections | Guidance only |

`PY001` reports a warning only for a bare call expression directly in the
module body. Calls in assignments, decorators, class bodies, function bodies,
and an `if __name__ == "__main__"` body are not inferred to be import-time
side effects. Intentional import-time registration can use the existing narrow
suppression syntax.

`PY004` reports one warning per public function or method with missing
parameter or return annotations. It exempts `self` and `cls`, private and
dunder definitions, nested functions, overloads and stubs, and test functions
using the same public-surface rules as docstring enforcement. It checks only
annotation presence; a configured type checker remains responsible for type
correctness.

`PY001` and `PY004` are ordinary high-confidence warnings and retain the
existing Stop-gate behavior. `PY002` is explicitly organizational guidance,
and `PY005` recommends comprehensions only for a pure, straightforward
one-pass map or filter; neither becomes a mechanical style preference.

## Documentation and packaging

Update the README and repository guidance with both new skills, the `SOLID*`
and `PY*` namespaces, checker boundaries, and non-blocking SOLID Stop behavior.
Bump the plugin feature version from `0.3.1` to `0.4.0` in both host manifests,
the Claude marketplace manifest, the package-consistency test, and the README
badge.

## Verification

Tests must demonstrate the red-green behavior of each supported incompatibility,
representative compatible overrides, unguarded top-level calls, and public
annotation gaps. Hook integration tests must show that a changed `SOLID03`
warning does not block stopping while existing blocking findings—including
`PY001` and `PY004`—still do. Final verification consists of the complete unit
suite, the explicit self-audit over `hooks`, `tests`, and `skills`, and Claude
plugin manifest validation.
