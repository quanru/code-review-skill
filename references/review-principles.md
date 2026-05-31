# Code Review Principles

Treat this document as the skill's primary standard, not as a loose checklist.

## Coding Style Standards

### 1. Code Organization and Refactoring Philosophy

**Extract functions to enable reuse**

- Extract duplicated code into helper functions with clear responsibilities.
- Example: extract `buildUsageInfo()` to replace repeated object construction.

**Simplify by removing unnecessary abstractions**

- Remove complex abstractions when a direct implementation works better.
- Example: simplify multiple conditionals into direct delegation.

**Large-scale refactoring**

- Do not avoid larger refactors when they materially improve the architecture.

### 2. Type Safety and TypeScript Usage

- Use `as` casts in specific, justified situations.
- Use `satisfies` for type checking.
- Use constant assertions with `as const`.
- Write complete JSDoc comments for public APIs.

### 3. Naming Standards

**Constants**

- Use `camelCase` for ordinary constants inside functions.
- Use descriptive, contextual names such as `defaultMaxUserImagesCount` and
  `defaultReplanningCycleLimit`.
- Use uppercase names for environment variables, such as
  `MIDSCENE_REPLANNING_CYCLE_LIMIT`.

**Functions**

- Use verb-based names: `buildUsageInfo`, `resolveReplanningCycleLimit`,
  `appendExecutionDump`.
- Give private methods meaningful contextual prefixes instead of underscores.
- Use descriptive parameter names: `usageData`, `modelConfigForPlanning`,
  `maxImageMessages`.

**Variables**

- Prefer descriptive names over abbreviations: `imageCount`, `executionDump`,
  `processedMessages`.
- Use context-aware names: `envReplanningCycleLimitRaw`,
  `envReplanningCycleLimit`.

### 4. Function Design Patterns

- Use optional parameters with defaults:
  `options?.maxImageMessages ?? this.maxUserImageMessages`.
- Use early returns for guard clauses.
- A reverse-then-reverse pattern can be appropriate when processing from the end.

### 5. Error Handling

- Use explicit error messages with context.
- Use dedicated error-handling functions or methods when appropriate.
- Combine try/catch with logging when the operation is recoverable.

### 6. Testing Patterns

- Provide comprehensive coverage, especially for edge cases and boundary
  conditions.
- Use descriptive test names.
- Use helper functions in tests.
- Keep expectations clear; add brief intent comments when needed.

### 7. Cleanup and Maintenance

- Delete unused code completely; do not comment it out.
- Merge related logic.
- When touching related code, fix nearby lint issues.

### 8. Configuration and Defaults

- Resolve defaults in layers: explicit options -> model-specific defaults ->
  environment variables/global defaults.
- Read environment variables with clear fallbacks.

### 9. Data Structure Patterns

- Use `WeakMap` when object association fits the problem.
- Use array spreads for immutable updates.

### 10. Commit Message Style

- Use conventional commits: `type(scope): description (#PR_number)`.
- Common types: `feat`, `fix`, `refactor`, `chore`, `docs`.
- Prefer explicit scopes: `core`, `web`, `android`, `ios`, `mcp`, and similar.

### 11. Code Style Preferences

- Use ternaries for simple conditions when they improve readability.
- Use optional chaining `?.` widely.
- Prefer nullish coalescing `??` over OR `||`.
- Use typed destructuring when it clarifies intent.

### 12. Documentation

- Update documentation along with code changes.
- If the project maintains documentation in multiple languages, keep the
  languages in sync.
- Make error messages helpful enough to diagnose the issue.

## Key Principles

1. **Prefer simplicity**: remove unnecessary abstractions and simplify whenever
   possible.
2. **Type safety**: use TypeScript features such as `satisfies`, `as const`, and
   precise types.
3. **Comprehensive tests**: cover edge cases with descriptive test names.
4. **Clean refactoring**: do not fear larger changes that improve architecture.
5. **Clear naming**: use contextual, descriptive variable and function names.
6. **Appropriate error handling**: centralize it when useful and keep messages
   clear.
7. **Immutable patterns**: prefer spreads and avoid accidental mutation.
8. **Layered defaults**: make configuration priority explicit.
9. **Documentation sync**: update docs and comments when behavior changes.
10. **Conventional commits**: keep commit messages scoped and standardized.

## High Cohesion and Low Coupling (Critical Review Area)

This skill includes its own cohesion/coupling review method. Do not rely on
another skill for it.

**High-cohesion principles**

- Each class or module should have one responsibility.
- Data and the behavior that operates on that data should live together.
- A class name should accurately describe everything the class does.

**Low-cohesion red flags**

- A class is hard to name, or its description keeps needing "and".
- Methods inside a class do not center on the same core state or domain object.
- Utility functions, formatting, cache logic, or queue logic are scattered in
  callers instead of living with the data or resource owner.

**Low-coupling principles**

- Communicate through interfaces rather than directly depending on
  implementations.
- A change in one class should not force changes in other classes.
- Dependencies should be injected, not hard-coded.

**High-coupling red flags**

- A caller knows the callee's internal state, cache strategy, serialization
  details, or temporary constraints.
- Changing one module often requires synchronized changes in neighboring
  modules.
- Dependencies are wired through hard-coded access, globals, or special-case
  branches, making replacement and testing difficult.

**Dedicated review workflow**

1. List the classes and modules touched by the change and their responsibilities.
2. For each responsibility, ask "why is it here?" and check whether a more natural
   data owner or boundary owner exists.
3. Identify coupling points between modules: method calls, shared data
   structures, state exposure, and serialization protocols.
4. Prefer structural suggestions such as responsibility movement, interface
   narrowing, and state encapsulation over local conditional patches.

**Component inventory template**

```markdown
| Component | Current Responsibilities | Natural Owner? | Coupling Notes |
|-----------|--------------------------|----------------|----------------|
| Agent | Task execution, report triggering | Partly no | Should not manage ReportGenerator's internal queue |
| ReportGenerator | File writes, template handling | Yes | Should hide its own write strategy |
```

**Checklist**

- Is logic in the wrong place, such as utilities living in a caller rather than
  the data owner?
- Does one class know another class's implementation details?
- Are serialization and deserialization cohesive inside the data class?
- Are internal queues, caches, or state-management details exposed externally?
- Is the code using an ask pattern, where a caller queries internal state and then
  assembles business logic externally, instead of a tell pattern?
- If changing A always requires changing B, could that coupling be removed through
  an interface or responsibility split?

**Real cases**

*Case 1: queue management should be cohesive*

```typescript
// Bad: Agent manages ReportGenerator's write queue.
class Agent {
  private writeQueue: Promise<void>;
  private scheduleWrite() { /* queue logic */ }
}

// Good: ReportGenerator manages its own queue.
class ReportGenerator {
  private writeQueue: Promise<void>;
  onDumpUpdate() { /* queue logic */ }
  flush() { await this.writeQueue; }
}
```

*Case 2: data processing should live with the data owner*

```typescript
// Bad: ReportGenerator knows how to process base64.
class ReportGenerator {
  private stripBase64Prefix(base64: string) { /* ... */ }
}

// Good: ScreenshotItem provides its own data conversion.
class ScreenshotItem {
  get rawBase64(): string { /* ... */ }
}
```

*Case 3: serialization format should be decided by the data class*

```typescript
// Bad: external code walks objects to replace serialization format.
const replaceScreenshotRefs = (obj: Record<string, unknown>) => {
  if ('$screenshot' in obj) return { base64: path };
};

// Good: the data class decides its own serialization format.
class ScreenshotItem {
  markPersistedToPath(path: string) {
    this._persistedAs = { base64: path };
  }

  toSerializable() {
    return this._persistedAs ?? { $screenshot: this._id };
  }
}
```

**Additional judgment**

- Focus first on whether responsibilities are clear, not just on class or file
  size.
- If a caller assembles too many low-level details, the encapsulation boundary is
  usually wrong.
- Prefer intent-shaped interfaces over exposing internal details for external
  handling.
- Explicit dependencies are more important than implicit shared state; inject
  dependencies instead of hard-coding them when possible.

## Elegant Implementation (Critical Review Area)

Elegance does not mean cleverness. It means solving the problem with fewer
concepts, fewer branches, and a clearer main path. Review whether the
implementation is natural, direct, and extensible instead of being assembled from
patch-like conditions.

**Judgment principles**

- The main path should be readable at a glance. Special cases should be contained
  at boundaries, not scattered through several branches.
- An abstraction is valid only when it truly reduces complexity. If it merely
  moves complexity elsewhere, it is a bad abstraction.
- Callers should express business intent, not assemble low-level details.
- Problems that can be solved through data structures, type design, or
  responsibility boundaries should not degrade into flags, temporary state, and
  nested conditionals.
- A change that only solves the current case while making future evolution harder
  is not elegant.

**Checklist**

- Could a better interface or data structure replace newly added `if/else`
  branches, flags, or special cases?
- Is the implementation adding checks in many places to support one new scenario?
- Did it add helpers, wrappers, intermediate variables, or config options that
  only serve one path?
- Does the caller know details it should not know?
- Is the default path still the shortest, most direct, and easiest to understand?
- Is the same intent expressed in multiple places with different code shapes?
- Will this change make future deletion, replacement, or extension cheaper or
  more expensive?

**Real cases**

The following examples come from the Midscene repository.

*Case 1: normalize input at the boundary to avoid repeated branches in the main flow*

`packages/core/src/common.ts` uses `normalizeBboxInput` to unify four input shapes
into one, so `adaptBbox` consumes a normalized structure:

```typescript
function normalizeBboxInput(
  bbox: AdaptBboxInput,
): number[] | string[] | string {
  if (Array.isArray(bbox)) {
    if (Array.isArray(bbox[0])) {
      return bbox[0] as number[] | string[];
    }
    return bbox as number[] | string[];
  }
  return bbox as string;
}

export function adaptBbox(
  bbox: AdaptBboxInput,
  width: number,
  height: number,
  modelFamily: TModelFamily | undefined,
): [number, number, number, number] {
  const normalizedBbox = normalizeBboxInput(bbox);
  // The rest of the flow consumes normalizedBbox.
}
```

`packages/core/src/yaml/utils.ts` unwraps `{ prompt: 'counter' }` at the boundary
inside `buildDetailedLocateParam`, preventing downstream code from creating a
double nesting such as `{ prompt: { prompt: 'counter' } }`:

```typescript
let normalizedLocatePrompt: TUserPrompt = locatePrompt;
if (
  typeof locatePrompt === 'object' &&
  locatePrompt !== null &&
  'prompt' in locatePrompt
) {
  const { prompt: innerPrompt, ...rest } = locatePrompt;
  const hasMultimodalFields = Object.keys(rest).length > 0;
  normalizedLocatePrompt = hasMultimodalFields ? locatePrompt : innerPrompt;
}
```

*Case 2: use explicit modeling instead of boolean switches*

`packages/playground/src/platform.ts` uses a union type instead of several boolean
flags:

```typescript
// Bad: showScreenshot: boolean, showMjpeg: boolean, showScrcpy: boolean ...
// Good: the mode enters the type system, so illegal combinations are impossible.
export type PlaygroundPreviewKind =
  | 'none'
  | 'screenshot'
  | 'mjpeg'
  | 'scrcpy'
  | 'custom';

export interface PlaygroundPreviewCapability {
  kind: PlaygroundPreviewKind;
  label?: string;
  live?: boolean;
}
```

*Case 3: contain special cases at the boundary instead of polluting the main flow*

`packages/core/src/ai-model/auto-glm/util.ts` keeps model-family logic in
predicate functions. The main flow expresses intent through `isUITars(modelFamily)`
instead of scattering repeated comparisons:

```typescript
export function isAutoGLM(modelFamily: TModelFamily | undefined): boolean {
  return modelFamily === 'auto-glm' || modelFamily === 'auto-glm-multilingual';
}

export function isUITars(modelFamily: TModelFamily | undefined): boolean {
  return (
    modelFamily === 'vlm-ui-tars' ||
    modelFamily === 'vlm-ui-tars-doubao' ||
    modelFamily === 'vlm-ui-tars-doubao-1.5'
  );
}
```

`packages/shared/src/env/parse-model-config.ts` contains intent-to-key mapping in
`KEYS_MAP`, so the main flow can index by intent:

```typescript
const KEYS_MAP: Record<TIntent, TModelConfigKeys> = {
  insight: INSIGHT_MODEL_CONFIG_KEYS,
  planning: PLANNING_MODEL_CONFIG_KEYS,
  default: DEFAULT_MODEL_CONFIG_KEYS,
} as const;
```

*Case 4: prefer data-structure-driven logic to field-by-field conditionals*

`packages/core/src/yaml/utils.ts` uses a `Record` to list multimodal field names
and extracts them by iteration instead of writing one conditional per field:

```typescript
const multimodalLocateOptionFieldMap: Record<keyof TMultimodalPrompt, true> = {
  images: true,
  convertHttpImage2Base64: true,
};

const multimodalLocateOptionKeys = Object.keys(
  multimodalLocateOptionFieldMap,
) as Array<keyof TMultimodalPrompt>;

function extractMultimodalPrompt(
  opt?: LocateOption,
): Partial<TMultimodalPrompt> | undefined {
  if (typeof opt !== 'object' || opt === null) return undefined;

  const entries = multimodalLocateOptionKeys
    .map((key) => [key, opt[key]] as const)
    .filter(([, value]) => value !== undefined);

  return entries.length
    ? (Object.fromEntries(entries) as Partial<TMultimodalPrompt>)
    : undefined;
}
```

Adding a new multimodal field requires one line in
`multimodalLocateOptionFieldMap`, not a change to the extraction algorithm.

*Case 5: layered default resolution*

`packages/core/src/utils.ts` resolves cache configuration by priority: explicit
parameter -> environment variable -> no cache:

```typescript
export function processCacheConfig(
  cache: Cache | undefined,
  cacheId: string,
): Cache | undefined {
  if (cache !== undefined) {
    if (cache === false) return undefined;
    if (cache === true) return { id: cacheId };
    if (typeof cache === 'object' && cache !== null) {
      if (!cache.id) return { ...cache, id: cacheId };
      return cache;
    }
  }

  const envEnabled = globalConfigManager.getEnvConfigInBoolean(MIDSCENE_CACHE);
  if (envEnabled && cacheId) return { id: cacheId };

  return undefined;
}
```

## Common Antipatterns (Critical Review Area)

Actively check for these antipatterns during review.

### Antipattern 1: shape changes in parameter passing chains

Data unexpectedly changes shape while moving through layers: objects get wrapped
again, nesting increases, or fields are spread and lost.

```typescript
// Bad: locatePrompt is already { prompt: 'counter' }, so this double-wraps it.
const param = { prompt: locatePrompt }; // -> { prompt: { prompt: 'counter' } }

// Good: inspect the input shape and avoid repeated wrapping.
const param = typeof locatePrompt === 'string'
  ? { prompt: locatePrompt }
  : locatePrompt;
```

**Review method**: trace key parameters from construction to consumption and
verify each layer's shape expectation.

### Antipattern 2: bypassing existing shared utilities

The repository already has shared utilities for logging, temporary directories,
data transforms, and similar concerns, but new code reimplements them locally.

**Review method**: when you see `console.log`, hand-rolled temp paths, or repeated
data-processing logic, search shared modules for an equivalent utility. Logging
should use the project's logger rather than bare `console.log/warn/error`.

### Antipattern 3: silent fallback in error handling

Unexpected situations are swallowed silently, delaying failure until the problem
is harder to diagnose.

```typescript
// Bad: silent fallback.
try { doSomething(); } catch { return null; }

// Good: fail fast and preserve a diagnosis path.
try { doSomething(); } catch (e) {
  throw new Error(`doSomething failed: ${e.message}`, { cause: e });
}
```

**Review method**: search for empty catch blocks, unconditional `null`/`undefined`
returns, and conditionals without a meaningful else branch. When capability is
missing, warn users rather than hiding it.

### Antipattern 4: patch-like type design

Each extension adds another special-case field to an existing type instead of
converging on a stable type hierarchy.

```typescript
// Bad: fields keep accumulating.
interface Config {
  timeout?: number;
  resolvedTimeout?: number;
  planningTimeout?: number;
}

// Good: keep it flat and direct, or use a base type when there is a real subtype.
interface ModelConfig {
  timeout?: number;
}
interface PlanningModelConfig extends ModelConfig { /* planning-only fields */ }
```

**Review method**: check whether new optional fields have clear default semantics
and whether they introduce unnecessary intermediate concepts.

### Antipattern 5: changing one peer module but forgetting the rest

In a monorepo or multi-platform project, a change covers one module but misses
parallel modules.

**Review method**: identify peer modules such as platform adapters, SDK wrappers,
or entrypoints, and decide whether the change should be synchronized.

### Antipattern 6: new capability pollutes default behavior

A new feature changes behavior when no new parameter is provided.

**Review method**: verify that the path without the new parameter remains the
same as before. New behavior should take a separate branch while preserving the
default path.

### Antipattern 7: incomplete configuration documentation

A new configuration option documents only its name and type, without default
value, use case, or misuse guidance.

**Review method**: each new config option should document its default, when to
change it, and what can go wrong if it is set incorrectly.

### Antipattern 8: stale references after rename

After renaming variables, functions, classes, or files, old names remain in docs,
types, tests, or comments.

**Review method**: search globally for old names and confirm code, docs, tests,
and comments are all updated.

### Antipattern 9: simplification removes testability

An abstraction layer is removed, but the testable interface disappears with it,
leaving behavior covered only through integration tests.

**Review method**: if the removed abstraction had tests, verify that equivalent
behavior remains unit-testable after simplification.

### Antipattern 10: public API rename without transition

A public API is renamed directly with no deprecation warning or alias, breaking
downstream consumers silently.

**Review method**: when exported names, method names, or config keys change,
check for a transition path such as a deprecated alias plus warning.

### Antipattern 11: new dependency without license check

A new dependency is added without checking whether its open-source license is
compatible with the project.

**Review method**: new dependencies and devDependencies should have clear,
compatible licenses.

### Antipattern 12: circular dependencies

New imports form an A -> B -> A cycle, directly or indirectly.

**Review method**: inspect new imports for direct and indirect cycles.

### Antipattern 13: shared module changes without downstream validation

A shared module's export or interface changes, but downstream consumers are not
built or tested.

**Review method**: when a shared module changes, run builds or tests for projects
that depend on it.

### Antipattern 14: incomplete cleanup after add-then-remove iteration

Experimental code is removed, but dead code, unused imports, orphaned config, or
fixtures remain.

**Review method**: after removing a feature, search for related imports, type
references, config keys, and test fixtures.

## Review Task

1. Identify the current code change.
   - If the user specified files, review those files.
   - Otherwise inspect the current PR diff first and review that diff if
     available.
   - If PR diff is unavailable, inspect the directly available repository diff.
   - Ask what to review only when no reviewable scope can be determined.
2. Check item by item.
   - Review against every standard above; do not stop at correctness.
   - Mark violations with file paths and line numbers.
   - Provide concrete improvement suggestions and short code examples when
     useful.
3. Run the high-cohesion/low-coupling pass.
   - Check whether responsibilities live in the right classes.
   - Check whether logic should move to the data owner.
   - Check whether module coupling is too tight.
   - Use the real cases above as comparison points.
4. Run the elegant-implementation pass.
   - Check whether the implementation advances through patch-like special cases.
   - Check whether the main path is clear and special cases are contained at
     boundaries.
   - Check whether better data structures, type design, or interface narrowing
     could reduce branches.
   - Use the real cases above as comparison points.
5. Run the antipattern pass.
   - Compare the current diff against the common antipatterns above.
   - Focus especially on parameter-shape changes, bypassed shared utilities,
     silent fallback, cross-module consistency, and configuration documentation.
6. Provide an overall assessment.
   - Rate the code quality, such as excellent, good, or needs improvement.
   - List the top 3-5 improvement points.
   - Also call out good code that follows the standards.

## Output Requirements

Because higher-level Codex review instructions also require findings first, use
this order to keep behavior consistent across environments:

1. **Findings**
   - List high-priority issues first, then medium-priority issues.
   - Include file paths and line numbers.
   - Emphasize behavioral risk, architecture risk, and potential regressions.
2. **What works well**
   - Only after the main issues.
3. **Overall assessment**
   - Provide a brief conclusion, optionally by dimension.
4. **Priority improvements**
   - Give the most important 3-5 actions.

Recommended structure:

```markdown
## Code Review Report

### Scope
[List reviewed files]

### Findings

#### Cohesion and Coupling Issues
- [Issue: file:line] Description + why cohesion/coupling is weak + suggestion

#### Other High-Priority Issues
- [Issue: file:line] Description + risk + suggestion

#### Medium-Priority Issues
- [Issue: file:line] Description + suggestion

### What Works Well
- [Good example that follows the standard]

### Overall Assessment
- Cohesion: ...
- Coupling: ...
- Code style: ...
- Overall: ...

### Priority Improvements
1. [Most important improvement]
2. [Second most important improvement]
3. [Other improvement]
```

If no high-priority issue is found, state that clearly and include remaining risk
or test gaps.
