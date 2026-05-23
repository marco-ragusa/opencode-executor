# opencode-executor

A Claude Code skill that delegates code generation to OpenCode while Claude handles planning, task specification, and diff review.

## Motivation

Frontier models (Claude Sonnet, Opus) are efficient at reasoning over context, planning changes, and reviewing diffs. They are not cost-efficient for iterative code generation, which involves repeated context loads, multiple correction rounds, and noisy intermediate output that accumulates in the context window.

OpenCode is a terminal-based AI coding agent that accepts a text prompt and runs against any backend: a local model via Ollama, a mid-tier API model, or any OpenAI-compatible endpoint. Delegating the implementation loop to a cheaper backend concentrates frontier spending on the parts where it matters (planning, constraint specification, diff review) and offloads mechanical code generation to a lower-cost executor.

A secondary benefit is auditability. Every delegation goes through a task file written to `opencode_tasks/YYYYMMDD-<slug>.txt`. The task file is explicit, repeatable, and scoped: it names the exact files to modify, the required changes, and a list of forbidden behaviors. This forces the orchestrator to be precise before any code is written, which surfaces ambiguities early.

## How It Works

The workflow follows a fixed sequence:

1. Claude inspects the target files and understands the request.
2. Claude captures a baseline: `BASELINE=$(git rev-parse HEAD)`, written to `opencode_tasks/baseline.txt`.
3. Claude writes a task file with explicit target files, required changes, constraints, and a "do not" list.
4. OpenCode runs the task: `TASK=$(cat "opencode_tasks/YYYYMMDD-<slug>.txt") && opencode run "$TASK"`.
5. Claude reviews the diff against the baseline: `git diff "$(cat opencode_tasks/baseline.txt)"`.
6. Claude validates with targeted commands (pytest, npm test, cargo test, etc.).
7. If the patch is wrong, Claude writes a focused corrective task and repeats from step 4.

The baseline persists across correction iterations. Every subsequent `git diff "$(cat opencode_tasks/baseline.txt)"` shows the cumulative OpenCode contribution since capture, regardless of whether OpenCode staged, committed, or left changes unstaged.

## Token Cost Estimate

The following estimate applies to a mid-size feature: one function added or modified, with tests. It assumes the frontier model is Claude Sonnet and the OpenCode backend is a local model or a cheaper API model.

**Without delegation (Claude handles everything):**

| Phase | Approximate tokens |
|---|---|
| File inspection | 4,000 - 8,000 |
| Planning and reasoning | 2,000 - 4,000 |
| Implementation iterations (3-5 rounds) | 30,000 - 60,000 |
| Review | 2,000 - 3,000 |
| Total | 38,000 - 75,000 |

**With delegation:**

| Phase | Claude (frontier) | OpenCode backend |
|---|---|---|
| File inspection | 4,000 - 8,000 | 0 |
| Task file and planning | 2,000 - 4,000 | 0 |
| Implementation iterations | 0 | 20,000 - 50,000 |
| Diff review and validation | 3,000 - 6,000 | 0 |
| Total | 9,000 - 18,000 | 20,000 - 50,000 |

Estimated frontier token reduction: 70-80% per task.

If the OpenCode backend is a local model, the implementation phase costs nothing. If it is a cheaper API model, the cost scales with that model's pricing. As a reference, `claude-haiku-4-5` is priced roughly 10x lower than `claude-sonnet-4-6` per input token. The savings scale with task complexity: simple single-file patches see the smallest absolute gain; multi-file changes with several correction rounds see the largest.

These numbers assume 3-5 implementation iterations, which is typical when delegating to a smaller model that requires more corrective guidance. A higher iteration count increases backend token usage but leaves frontier usage flat, since Claude only sees the final diff of each round.

## Requirements

- Claude Code (CLI or desktop app)
- OpenCode CLI available in PATH (`opencode --version` should succeed)
- Git repository with at least one commit in the target project

## Skill Location

```
skills/opencode-executor/SKILL.md
```

The skill triggers when you mention OpenCode explicitly: "use opencode to add X", "opencode task: fix Y", "delegate to opencode".
