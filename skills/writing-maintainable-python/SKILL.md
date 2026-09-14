---
name: writing-maintainable-python
description: Use when writing or reviewing Python entry points, import-time execution, public type annotations, oversized or multi-purpose functions, or loop-versus-comprehension readability.
---

# Writing Maintainable Python

Make execution boundaries and public contracts explicit. Prefer structures
whose intent is visible on import, in an editor, and to a static checker.

## Practices

| ID | Practice | Enforcement |
|---|---|---|
| `PY001` | Guard executable module code | Bare top-level call expressions are warnings |
| `PY002` | Define `main()` | Organizational guidance |
| `PY003` | Keep functions small and single-purpose | `POT04` checks length; responsibility needs review |
| `PY004` | Annotate public interfaces | Missing parameter or return annotations are warnings |
| `PY005` | Use simple comprehensions when clearer | Readability guidance |

## Entrypoints

Importing a module executes its top-level statements. Put an executable
workflow in a typed `main()` and call it only under the standard guard:

```python
def main() -> None:
    client = connect()
    process(client)


if __name__ == "__main__":
    main()
```

`PY001` deliberately checks only bare calls directly in the module body. An
intentional import-time registration call needs an adjacent, specific
suppression. Calls used in assignments or decorators are not inferred to be
side effects.

## Functions and types

Extract a function when it owns a separately nameable rule, phase, or reuse
boundary—not merely to reduce line count. `POT04` limits a function to 60 code
lines; `SOLID01` asks whether it has multiple demonstrated reasons to change.

Annotate every public parameter and return value, including `*args` and
`**kwargs`. `self` and `cls` are implicit. Annotations document the contract
and enable editor feedback, but Python does not enforce them at runtime; run
the project’s configured checker such as mypy or pyright. `PY004` checks
presence only, not correctness.

## Comprehensions

Prefer a list comprehension for a pure, straightforward one-pass mapping or
filter:

```python
long_names = [name for name in names if len(name) > 7]
```

Use an ordinary loop when the body has side effects, multiple branches,
multiple nested clauses, error handling, or needs an explanatory name. A
comprehension that merely compresses complexity is less maintainable, and it
does not establish an input or resource bound.

## Common mistakes

- A guard around several procedural statements: move their orchestration into `main()`.
- Tiny forwarding helpers with no independent concept: keep cohesive code together.
- Annotations with no type-check command: add the project’s checker instead of assuming runtime enforcement.
- Nested comprehensions chosen for brevity: expand them when the reader must simulate control flow.

Run `review-code-quality` after changes. These practices improve explicitness;
they do not replace tests, runtime validation, or architectural review.
