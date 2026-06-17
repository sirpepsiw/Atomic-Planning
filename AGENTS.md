# Permanent Workflow Rules — ALWAYS ACTIVE

## Atomic Task Decomposition (MANDATORY)

Every plan MUST be decomposed into atomic tasks BEFORE any implementation.
This applies regardless of which model is planning.

### Rules:
- One task per file
- Max 200 lines of changes OR ~200 words of SPECS per task (whichever hits first)
- Each task must be independently verifiable with concrete, testable criteria
- Tasks declare DEPENDS_ON for explicit dependencies (conservative: when in doubt, declare)
- Tasks declare which files they CREATE and MODIFY (FILES key in task header)
- Two tasks touching the same file → sequential, never parallel
- Shared files (routes, configs, package.json, type definitions) → dedicated integration task in final wave
- Feature tasks produce self-contained modules; integration task wires them in
- For HTML/documents: SCAFFOLD first (structure + CSS), then fill content per section

### Task format:

```
## FILE: <exact path>

### TASK: <one-line description>
FILES: [create: <file1>, modify: <file2>]
DEPENDS_ON: <task-id>

#### SPECS (EARS format — WHEN / THE / SHALL / SHALL NOT / IF)
- WHEN <trigger> THE <component> SHALL <expected behavior>
- IF <condition> THEN <behavior>
- SHALL NOT <forbidden behavior>
- <value must be explicit: "HTTP 200", "red (#ff0000)", "3 seconds">
- <"Build a component" = BAD. "WHEN user clicks THE button SHALL show loading state" = GOOD>

#### CONSTRAINTS
- <framework, patterns to follow>
```

### Short form (no FILES/DEPENDS_ON when trivial singleton task):

```
## FILE: <exact path>

### TASK: <one-line description>

#### SPECS (EARS format)
- WHEN <trigger> THE <component> SHALL <behavior>
- IF <condition> THEN <alternative behavior>
- SHALL NOT <forbidden>

#### CONSTRAINTS
- <framework, patterns to follow>
```

## Lightweight Validation Gates (MANDATORY)

After executor returns the result, ALWAYS run these 3 checks BEFORE applying:

1. **File integrity**: Was the target file created or modified correctly? (exists, non-empty, valid format)
2. **Specs conformance**: Do key specs hold? (spot-check 2-3 critical requirements)
3. **Syntax check**: Does the code parse cleanly without obvious errors?

If any check fails:
- First failure: request fix from executor with specific feedback (max 2 retries)
- After 2 retries: re-plan from scratch with corrected specs
- If re-plan also fails: escalate to user

This prevents bad output from compounding into downstream tasks.

## Execution Decision (MANDATORY)

After producing the plan, evaluate delegation convenience using these heuristics, then ask with recommendation.

### Delegation recommended (use executor):
- ≤1 file, ≤50 lines of output code
- Well-defined EARS specs, no design judgment needed
- CRUD, styling, boilerplate, or mechanical changes
- Low risk of re-interpretation ambiguity

### Delegation NOT recommended (execute directly):
- Multi-file refactors with cross-cutting logic
- Bug with unclear root cause
- Requires deep codebase domain knowledge beyond specs
- Task would likely need ≥3 validation retries

### Ask with recommendation:

> "¿Delegar a Qwen local? [✅ Sí | ❌ No es recomendable]
> Razón: <one-line justification>"

- If user delegates → load skill → purge → executor → validate → apply → report
- If user says no → wait for instructions
