# code-review

Architecture- and maintainability-focused code review skill for coding agents.

This skill emphasizes responsibility boundaries, high cohesion and low coupling,
TypeScript rigor, naming, tests, configuration defaults, documentation sync, and
removing unnecessary abstractions.

## Features

- Findings-first review output
- Diff-first scope discovery
- Dedicated checks for cohesion, coupling, elegance, TypeScript API design, and
  common maintainability antipatterns
- Concrete file and line references with actionable improvement suggestions

## Installation

```bash
npx skills add quanru/code-review
```

For Codex project-local installation:

```bash
npx skills add quanru/code-review -a codex
```

## Usage

In your coding agent, say:

```text
code review this change
```

```text
review the current diff for architecture and maintainability risks
```

```text
run a code-review pass on src/foo.ts
```

## License

MIT
