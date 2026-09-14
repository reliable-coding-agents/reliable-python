---
name: review-code-quality
description: Use when auditing Python source, diffs, pull requests, or refactors for reliability, maintainability, SOLID, explicit program structure, or type-contract risks.
---

# Review Code Quality

Produce an evidence-based review of the changed scope. Separate mechanical
rule violations from heuristic design smells and avoid speculative findings.

## Review workflow

1. Read `../using-power-of-ten/references/power-of-ten.md`,
   `../using-power-of-ten/references/code-smells.md`,
   `../applying-solid-principles/SKILL.md`, and
   `../writing-maintainable-python/SKILL.md`.
2. Read repository guidance and identify the requested change, affected public
   behavior, and changed lines. Review the diff first; open surrounding code
   only as needed to prove a finding.
3. Run the bundled checker from the repository root:

   ```bash
   python3 <skill-directory>/scripts/audit_python.py --git-diff
   ```

   For explicit files or directories, replace `--git-diff` with their paths.
   Add `--docstring-style google|rest|any` when the project does not use the
   NumPy default. The checker covers only high-signal, statically detectable
   cases; it does not replace the semantic pass.
4. Trace the ten rules in order. For Rules 3, 8, and 9, label conclusions as
   Python-profile findings rather than literal C-rule compliance.
5. Review `PY001`-`PY005`: import safety, explicit entry-point assembly,
   function responsibility, public annotations, and comprehension readability.
6. Review `SOLID01`-`SOLID05` against demonstrated clients, extension points,
   substitution contracts, interface use, and dependency direction. Treat
   `SOLID03` checker output as evidence, not proof of the other principles.
7. Review the touched design against all five smell families. Report a smell
   only when the code shows the catalog's signal and a concrete maintenance or
   correctness cost.
8. Check the documentation convention on changed public units (`DOC01`-`DOC03`).
   Read `../writing-docstrings/SKILL.md` before reporting a docstring finding.
   A summary-only docstring satisfies every supported style and is not a
   finding on its own.
9. Run the repository's own formatter, linter, type checker, and focused tests.
   Do not substitute the bundled checker for project validation.
10. If asked to fix findings, make the smallest behavior-preserving change,
   rerun checks, and re-review the resulting diff.

## Finding standard

Report only findings that are actionable and caused or exposed by the changed
scope. Use:

```text
[severity] ID — concise title
location: path:line
evidence: what the code demonstrably does
impact: concrete failure or maintenance cost
remedy: smallest safe correction
```

Use `error` for a clear rule violation or correctness risk, `warning` for a
confirmed smell with material cost, and `note` for an explicit, justified
tradeoff. Do not report style preferences, existing unrelated debt, or a smell
name without evidence.

If no findings remain, say so and list the checks actually run. Never imply
that this review proves safety or exhaustively detects every smell.

## Suppressions

Honor `# quality: ignore[ID] - rationale` only when it is on the finding line or
immediately above the relevant statement and contains a specific rationale.
Challenge broad, stale, or circular rationales. Prefer correcting code over
adding a suppression.
