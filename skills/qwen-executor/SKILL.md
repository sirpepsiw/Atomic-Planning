---
name: qwen-executor
description: Delegates code execution to a local Qwen 7B model via Ollama. DeepSeek specs, Qwen generates from specs, DeepSeek applies. Saves output tokens.
---

## Delegation Protocol (MANDATORY)

After generating specification-based plan, delegate to the executor subagent.
NEVER implement the plan yourself.

Workflow: read files → generate specs (EARS format) → purge RAM → call executor with specs → validate output (3 gates) → apply result → report savings

The executor runs locally at zero API cost. Every token Qwen generates is one token DeepSeek does NOT bill.

## RAM Purge Before Delegation (MANDATORY)

Before calling the executor subagent, run:

```bash
sudo purge
```

This clears inactive memory pages, freeing ~5-6 GB RAM for Qwen 8.1 GB model.
With `NOPASSWD` configured in `/etc/sudoers.d/purge`, no password prompt occurs.

### Linux equivalent:

```bash
sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
```

Configure sudoers: `username ALL=(ALL) NOPASSWD: /usr/bin/tee`

Task decomposition follows AGENTS.md rules (atomic, 1 file per task, FILES/DEPENDS_ON headers).

## Plan Format for Qwen — SPEC-BASED (NOT copy)

CRITICAL: Do NOT include the full output code in the plan. LLMs reinterpret copied
code. Instead, provide SPECIFICATIONS that Qwen generates code FROM.

### For editing existing file:

```
## FILE: <exact relative path from repo root>

### TASK: <one-line description>
FILES: [modify: <file>]
DEPENDS_ON: <task-id>

#### EXISTING CODE
```<language>
<current complete file content>
```

#### SPECS (EARS format)
- WHEN <condition> THE <component> SHALL <expected behavior>
- IF <edge case> THEN <alternative>
- SHALL NOT <forbidden>

#### CONSTRAINTS
- Must preserve all existing imports
- Must follow existing naming conventions
- Do NOT add comments unless specified
```

### For creating new file:

```
## FILE: <exact relative path from repo root>

### TASK: <one-line description>
FILES: [create: <file>]
DEPENDS_ON: <task-id>

#### SPECS (EARS format)
- WHEN <trigger> THE <component> SHALL <behavior>
- IF <condition> THEN <alternative>
- SHALL NOT <forbidden>

#### CONSTRAINTS
- Language/framework: <language, version>
- Must follow same patterns as: <reference file path>
- Output only the file content, no explanations
```

### Key Rules for Plan Design

1. SPECS must use EARS format (WHEN/THE/SHALL/SHALL NOT/IF) for unambiguous requirements
2. Do NOT include full output code — only specs
3. For edits, include EXISTING CODE verbatim so Qwen has context
4. Keep total plan under ~6K tokens (leaves ~2K for Qwen's output)
5. One file per task, atomic changes only

## Executor Rules

- Exact paths from repo root
- Max ~8K tokens per task (plan + code + response)
- Temp 0.1, context 16384, disable thinking mode
- After executor returns: run validation gates (file integrity → specs conformance → syntax check)
- If validation fails: request fix from executor (specific feedback, max 2 retries)
- After 2 retries: re-plan from scratch with corrected specs
- Never ask Qwen to debug its own output — fix via new SPECS or re-plan

## Token Savings Report (MANDATORY)

After executor completes:
1. Count chars in executor code output ÷ 4 = tokens saved
2. Savings = tokens × price per 1M tokens
3. Report: "Done! ~X tokens saved locally (~$Y)"
4. Track cumulative session savings
