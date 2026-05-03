---
name: parallel-plan
description: >
  Create a "plan of plans": decompose a large task into N self-contained sub-plans
  that can be executed in parallel by independent agents, with zero dependencies
  between them. Use when the user says things like "parallel plan", "plan paralelo",
  "plan de planes", "ejecutar en paralelo", "split into parallel tasks", "dividir en
  paralelo", or asks for a plan that fans out work across multiple agents.
argument-hint: "[task description]"
---

# Parallel Plan

Decompose a large task into a set of independent sub-plans that fresh agents can execute concurrently. The output is a meta-plan: a plan whose leaves are themselves plans.

## When this applies

- The task is large enough that a single sequential plan would take too long.
- The task has natural seams of independence (different modules, different files, different layers, different test suites, etc.).
- The user explicitly asks to parallelize, fan out, or split the work.

If the task is small, has tight ordering constraints, or all changes touch the same file, **do not force parallelism** — say so and propose a normal sequential plan instead.

## Core invariants

A sub-plan is "parallel-safe" only if **all** of these hold:

1. **No file overlap.** No two sub-plans edit the same file. (Reading the same file is fine.)
2. **No ordering dependency.** Sub-plan B does not need anything sub-plan A produces (no shared types, no shared exports, no shared schema migrations).
3. **No shared in-flight state.** No two sub-plans modify the same DB table, the same feature flag, the same config key, the same global constant.
4. **Self-contained context.** Each sub-plan can be handed to a fresh agent with zero memory of this conversation — it must include the why, the files in scope, the acceptance criteria, and any relevant constraints.

If you cannot satisfy these, the work is **not** parallelizable. Flag it and stop.

## Steps to execute

### 1. Understand the task

Read the user's request carefully. If `$ARGUMENTS` is provided, that is the task. Otherwise, infer from the conversation. If the scope is ambiguous (which module? which feature? how deep?), ask **one** clarifying question before proceeding — do not guess.

### 2. Map the work

Before decomposing, briefly survey the surface area:
- Which files / modules / layers are involved?
- What are the natural boundaries (per-module, per-screen, per-endpoint, per-test-suite, per-language, frontend vs backend, etc.)?
- Are there cross-cutting changes (shared types, shared config, schema) that **must** happen first or last?

Use the Explore agent or `grep`/`find` if you genuinely don't know the layout. Don't skip this step — bad decomposition is worse than no decomposition.

### 3. Decompose

Split the task into N sub-plans (typically 2–6). For each sub-plan, write:

```markdown
### Sub-plan N: <short title>

**Scope:** <one-sentence description of what this sub-plan does>

**Files in scope:**
- path/to/file1.ts (edit)
- path/to/file2.ts (create)
- path/to/dir/ (edits within)

**Context the agent needs:**
<everything a fresh agent needs to execute this without seeing the rest of the meta-plan: why this exists, relevant constraints from CLAUDE.md, conventions to follow, what NOT to touch>

**Steps:**
1. <concrete step>
2. <concrete step>
3. ...

**Acceptance criteria:**
- <observable check 1>
- <observable check 2>
- Tests pass: `<command>`
- Type check passes: `<command>`
```

### 4. Verify independence

Before presenting the meta-plan, **explicitly verify** the invariants:

- Build a table: for each file path mentioned, list which sub-plan(s) touch it. If any path appears in 2+ sub-plans, **the decomposition is broken** — merge those sub-plans or rethink the seams.
- Check for implicit shared state: shared types, shared exports, shared constants, shared migrations, shared feature flags. If two sub-plans both add a key to the same enum or both export from the same barrel file, that is a conflict.
- If you find conflicts you cannot eliminate, either (a) extract the shared piece into a **prerequisite step** to run sequentially before the parallel fan-out, or (b) accept that the work isn't fully parallelizable and split into phases.

### 5. Write the plan files to disk

Persist the meta-plan and each sub-plan as their own `.md` files **before** presenting anything in chat. Fresh agents will be pointed at these files by path, so they must exist on disk and be self-contained.

**Location:** `<repo-root>/.claude/plans/<YYYYMMDD-HHMM>-<task-slug>/` in the user's current working directory.

- `<task-slug>` is a kebab-case 2–4 word summary of the task (e.g. `tests-uncovered-modules`, `migrate-auth-middleware`).
- Use the user's current local time for the timestamp.
- If `.claude/plans/` does not yet exist, create it. The first time you create this directory, suggest the user add `.claude/plans/` to `.gitignore` (do not edit `.gitignore` yourself unless the user agrees).

**Files to write:**

- `meta.md` — the wrapper: overview, prerequisites, independence check table, convergence steps, and a list of sub-plan files with one-line summaries.
- `sub-plan-1.md`, `sub-plan-2.md`, … — one file per sub-plan, containing the sub-plan body **verbatim** as defined in Step 3. Each file must stand alone: a fresh agent reading only this file must have everything it needs (scope, files, context, steps, acceptance criteria).

**`meta.md` structure:**

```markdown
# Parallel Plan: <task title>

## Overview
<2–3 sentences: what we're doing and why splitting helps>

## Prerequisites (sequential, if any)
<steps that must happen before fan-out, or "none">

## Sub-plans
- [Sub-plan 1: <title>](./sub-plan-1.md) — <one-line summary>
- [Sub-plan 2: <title>](./sub-plan-2.md) — <one-line summary>
- ...

## Independence check
| File / resource | Sub-plan(s) touching it |
| --- | --- |
| ... | ... |
<confirm no row has more than one sub-plan>

## Convergence (sequential, if any)
<steps that must happen after all sub-plans finish: integration tests, final wiring, release notes — or "none">
```

### 6. Present the meta-plan summary

In chat, give the user a **brief** summary, not a full dump (the files on disk are the source of truth):

- One-paragraph overview.
- Bulleted list of sub-plans with their one-line summaries and file paths.
- The independence-check table (always — it's the verification artifact).
- Prerequisites and convergence, if any.
- The directory where everything was written.

Do not paste the full body of each sub-plan into chat — point at the file instead.

### 7. Offer to execute

After the summary, ask the user:

> Quieres que lance los N agentes en paralelo ahora, o prefieres revisar los archivos en `.claude/plans/<dir>/` primero?

If they say yes, spawn all sub-plan agents in **one message** with multiple `Agent` tool calls (parallel execution). Each agent's prompt should be:

> Read `.claude/plans/<dir>/sub-plan-N.md` and execute it. The file is self-contained — follow its scope, steps, and acceptance criteria. Do not modify files outside its declared scope.

Use `subagent_type: "general-purpose"` unless a more specific agent fits. Use `isolation: "worktree"` only if the user asked for it or if the changes are large enough to warrant isolation.

## Anti-patterns (do not do these)

- **Fake parallelism.** Splitting a task that has real ordering dependencies and pretending the sub-plans are independent. The agents will conflict on merge.
- **Splitting too fine.** 12 sub-plans of 5 minutes each is worse than 3 sub-plans of 20 minutes — agent spawn overhead and merge complexity dominate.
- **Vague sub-plans.** "Refactor the auth module" is not a sub-plan. "Replace `useOldAuth` with `useNewAuth` in these 7 files, update the import in each, run the auth tests" is.
- **Hidden context.** Writing sub-plans that reference "the plan above" or "as discussed" — fresh agents have no such context.
- **Skipping the independence table.** It's the single most important verification step. Always include it.
- **Dumping the full plan into chat instead of writing files.** Fresh agents can't be reliably handed inline content from the conversation — they need a path to read. Always write `.md` files to `.claude/plans/<dir>/` first.

## Output language

Match the user's language. If the user wrote in Spanish, write the plan in Spanish. If in English, in English.
