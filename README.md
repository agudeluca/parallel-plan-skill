# parallel-plan

A [Claude Code](https://claude.com/claude-code) skill that decomposes a large task into N self-contained sub-plans that fresh agents can execute in parallel, with zero dependencies between them.

The output is a **meta-plan**: a plan whose leaves are themselves plans.

## When to use it

Trigger phrases:
- "parallel plan" / "plan paralelo" / "plan de planes"
- "ejecutar en paralelo" / "split into parallel tasks" / "dividir en paralelo"
- Any request to fan out work across multiple agents

Good fits:
- Large tasks with natural seams of independence (per-module, per-screen, per-endpoint, frontend vs backend, etc.)
- Refactors that touch many files but in disjoint sets

Bad fits (the skill will say so and propose a sequential plan instead):
- Tight ordering constraints
- All changes touch the same file
- Small enough that agent spawn overhead dominates

## How it works

1. **Understand the task** — clarify scope if ambiguous (one question max).
2. **Map the work** — survey files, modules, layers, and natural boundaries.
3. **Decompose** — split into 2–6 sub-plans, each with scope, files, context, steps, and acceptance criteria.
4. **Verify independence** — build a file-to-sub-plan table. If any file appears in 2+ sub-plans, the decomposition is broken.
5. **Write plan files to disk** — persist `meta.md` + `sub-plan-N.md` to `.claude/plans/<timestamp>-<slug>/` in the repo. Each sub-plan file is self-contained so a fresh agent can be pointed at it by path.
6. **Present a brief summary** — chat shows overview + sub-plan list with file paths + independence table, not the full body of each sub-plan (the files on disk are the source of truth).
7. **Offer to execute** — on confirmation, spawn N agents in a single message via the `Agent` tool, each pointed at its sub-plan file.

## Core invariants

A sub-plan is parallel-safe only if **all** of these hold:

1. **No file overlap.** No two sub-plans edit the same file.
2. **No ordering dependency.** Sub-plan B does not need anything sub-plan A produces.
3. **No shared in-flight state.** No two sub-plans modify the same DB table, feature flag, config key, or global constant.
4. **Self-contained context.** Each sub-plan can be handed to a fresh agent with zero memory of the conversation.

If these can't be satisfied, the work isn't parallelizable — the skill flags it and stops.

## Installation

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/agudeluca/parallel-plan-skill.git ~/.claude/skills/parallel-plan
```

Claude Code will pick up the skill automatically. Invoke it by asking for a parallel plan in natural language, or via `/parallel-plan <task>`.

## Anti-patterns the skill avoids

- **Fake parallelism** — splitting work with real ordering dependencies and pretending the sub-plans are independent.
- **Splitting too fine** — 12 sub-plans of 5 minutes each is worse than 3 sub-plans of 20 minutes.
- **Vague sub-plans** — "Refactor the auth module" is not a sub-plan; "Replace `useOldAuth` with `useNewAuth` in these 7 files" is.
- **Hidden context** — sub-plans that reference "the plan above" fail because fresh agents have no such context.
- **Skipping the independence table** — the single most important verification step.

## Output language

The skill matches the user's language: Spanish in, Spanish out; English in, English out.
