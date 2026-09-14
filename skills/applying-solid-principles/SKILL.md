---
name: applying-solid-principles
description: Use when designing or reviewing Python class hierarchies, protocols, interfaces, service boundaries, or dependency direction for SOLID-related maintainability risks.
---

# Applying SOLID Principles in Python

Use SOLID for concrete design costs, not as five labels for every class. A
finding needs evidence that substitution, extension, ownership, client
coupling, or dependency direction makes change harder or behavior less reliable.

## Principles

| ID | Principle | Python review question |
|---|---|---|
| `SOLID01` | Single responsibility | Does this unit have more than one demonstrated reason to change? |
| `SOLID02` | Open/closed | Can a required variant be added at a stable boundary, or must central policy be repeatedly edited? |
| `SOLID03` | Liskov substitution | Does every subtype preserve calls and behavior promised by its base or protocol? |
| `SOLID04` | Interface segregation | Are clients forced to depend on operations they cannot or do not use? |
| `SOLID05` | Dependency inversion | Does policy depend on a narrow abstraction instead of a volatile concrete detail? |

Class size, method count, constructor count, `isinstance`, or use of a concrete
class is not proof by itself. Trace an actual caller, extension, change axis,
or disabled operation before reporting a principle violation.

## Mechanical `SOLID03` evidence

The bundled auditor compares public overrides only when a base is a directly
named class in the same module. It warns when an override:

- replaces concrete behavior with `NotImplementedError`;
- changes method/property or synchronous/asynchronous behavior;
- removes positional, keyword, `*args`, or `**kwargs` calls accepted by the base;
- makes an optional argument required or adds a required argument.

External bases, inferred types, custom descriptors, unsupported decorators,
postconditions, exception guarantees, and semantic invariants remain manual.
`SOLID03` warnings appear in direct audits but are advisory to the Stop hook.
Use `# quality: ignore[SOLID03] - rationale` only beside a proven false positive.

## Example

```python
class Store:
    def fetch(self, key: str, timeout: float | None = None) -> bytes:
        raise LookupError(key)


class RemoteStore(Store):
    # Compatible: preserves existing calls and only widens acceptance.
    def fetch(
        self,
        key: str,
        timeout: float | None = None,
        **options: object,
    ) -> bytes:
        return request(key, timeout=timeout, **options)
```

Making `timeout` required, adding a required `region`, returning an awaitable,
or rejecting the operation would make `RemoteStore` unsafe at a `Store`
boundary. Preserve the contract; if the subtype cannot, prefer composition or
a narrower `Protocol` that states what clients can actually rely on.

## Review sequence

1. Identify the clients and promises at the changed boundary.
2. Trace replacement calls and failure behavior for `SOLID03` first.
3. Name demonstrated change reasons for `SOLID01` and extension edits for `SOLID02`.
4. Check which operations each client needs for `SOLID04`.
5. Confirm policy-to-detail dependency direction for `SOLID05`.
6. Report one principle that best explains the smallest safe remedy; do not duplicate a code-smell finding.

Run `review-code-quality` for the mechanical audit and structured finding
format. Passing checks does not prove SOLID compliance or architectural quality.
