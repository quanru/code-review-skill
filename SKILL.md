---
name: code-review
description: Perform architecture- and maintainability-focused code reviews. Use when the user asks for code review, style review, refactoring feedback, or review of architecture, cohesion, coupling, TypeScript rigor, naming, tests, responsibility boundaries, or unnecessary abstractions.
---

# Code Review

This skill provides a review standard centered on architectural consistency,
maintainability, and engineering quality.

The goal is not to run a generic code review checklist. First determine the
review scope, then systematically inspect responsibility boundaries, abstraction
level, type design, tests, and documentation.

`references/review-principles.md` contains the dedicated high-cohesion and
low-coupling review method. Do not depend on any other skill for those rules.

## Workflow

1. Read [references/review-principles.md](references/review-principles.md)
   before reviewing.
2. Treat that reference as the primary review standard, not supplemental reading.
3. Determine the review scope:
   - If the user specified files, review those files.
   - Otherwise inspect the current PR diff first. If available, review that diff
     first and open surrounding context files only as needed.
   - If a PR diff is unavailable, inspect the directly available repository diff.
   - Ask what to review only when neither the user nor the repository provides a
     reviewable scope.
4. Review against the reference item by item. Do not limit the pass to
   correctness. Cover at least:
   - responsibility ownership
   - high cohesion and low coupling
   - unnecessary abstractions
   - TypeScript type design and public API documentation
   - naming, error handling, tests, default configuration, and documentation sync
5. For architecture-heavy changes, run the dedicated cohesion/coupling workflow,
   red-flag checks, and case comparisons from the reference.
6. Cite specific files and lines, and provide direct improvement suggestions.
   Include short code examples when useful.

## Output

- Lead with findings, ordered by severity. This is required.
- After findings, include positives, overall assessment, and priority
  improvements.
- Adapt to a user-requested format if needed, but preserve the core review
  dimensions from the reference.
- If no issue is found, say clearly that no high-priority issue was found and
  call out remaining risk or test gaps.
