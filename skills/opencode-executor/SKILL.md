---
name: opencode-executor
description: "Delegate code implementation to the OpenCode CLI. Use only when the user explicitly mentions OpenCode — phrases like 'use opencode', 'opencode task', 'delegate to opencode', 'run via opencode'. Do not invoke for general coding questions or generic planner/executor patterns that do not mention OpenCode."
---

# OpenCode Executor Workflow

You orchestrate code changes by delegating implementation to the OpenCode CLI. Your role is reasoning, planning, and review. OpenCode's role is patch generation. `git diff` against a captured baseline is the verification interface.

## Direct Editing Policy

Do not directly edit repository implementation files for normal implementation work. All product code changes go through OpenCode unless the user explicitly instructs otherwise.

You may directly: inspect files, write task files per Step 4, run read-only commands, run tests/linters/typecheckers/builds, inspect `git status`, inspect `git diff`.

The reason: bypassing OpenCode defeats the cost split and bypasses the deterministic task-file paper trail. If you find yourself wanting to "just fix it directly", write a task file instead — it forces you to be explicit, which is usually where bugs in your own reasoning surface anyway.

## Safety Policy for User Worktree

The user's working tree may contain unstaged edits, scratch files, notes, generated artifacts, or the task file itself. Preserve all of it.

These commands are forbidden unless the user explicitly approves them, by name, for this specific operation:

```text
git restore <path>
git restore .
git checkout -- <path>
git checkout .
git reset
git reset --hard
git clean
git clean -fd
git clean -fdx
git rebase
git merge --abort
git rebase --abort
git stash push
git stash pop
git stash apply
rm -rf <path>
```

Note: the baseline mechanism uses `git rev-parse HEAD` — a read-only operation that never modifies the worktree or the stash stack.

If unrelated files appear after a run, do not assume they came from OpenCode. Report them, ask before acting. The cost of one accidental cleanup (permanent data loss) outweighs every legitimate cleanup combined.

For full reasoning on each forbidden command and recovery options, see `references/git-safety.md`.

## The Standard Workflow

Execute these steps in order for every implementation task:

1. **Understand the request.** Restate the goal in one sentence. If you can't, ask the user.
2. **Inspect the repository directly.** Read the target files, surrounding functions/classes, imports/types, existing tests, related implementations. Identify project conventions and existing validation commands.
3. **Capture the baseline.** Run this block unconditionally at the start of every new task, even if `baseline.txt` already exists from a previous task in this session. The file is always overwritten with the current HEAD SHA. The only exception is a corrective iteration on the current task — in that case, skip to Step 4 and reuse the existing baseline.

   ```bash
   git --version >/dev/null 2>&1 || { echo "ERROR: git is required." >&2; exit 1; }
   mkdir -p opencode_tasks
   [ -f opencode_tasks/.gitignore ] || echo "*" > opencode_tasks/.gitignore
   BASELINE=$(git rev-parse HEAD 2>/dev/null) || { echo "ERROR: repo has no commits — make at least one commit before using this skill." >&2; exit 1; }
   echo "$BASELINE" > opencode_tasks/baseline.txt
   git ls-files --others --exclude-standard | LC_ALL=C sort > opencode_tasks/untracked_before.txt
   ```

   `opencode_tasks/baseline.txt` holds the HEAD SHA at the moment baseline was captured. `git diff` against this SHA shows all changes introduced by OpenCode, whether left unstaged or committed. The untracked snapshot lets you distinguish "files OpenCode created" from "files the user already had untracked". The `.gitignore` in `opencode_tasks/` hides all skill artifacts from the repo's `git status`.

4. **Write the OpenCode task.** Choose a short slug describing the task (e.g., `add-retry`, `fix-auth-header`). Create the file:

   ```bash
   cat > "opencode_tasks/$(date +%Y%m%d)-<slug>.txt" <<'EOF'
   <write task content here — see OpenCode Task File Format section below>
   EOF
   ```

   Note the exact path of the file you just created — Step 5 references it directly. Be concrete: explicit files, explicit changes, explicit constraints, explicit "do not" list. Vague tasks produce vague patches, especially with small local models. See `references/examples.md` for good/bad examples.

5. **Run OpenCode**:

   ```bash
   TASK=$(cat "opencode_tasks/YYYYMMDD-<slug>.txt") && opencode run "$TASK"
   ```

   Replace `YYYYMMDD-<slug>` with the exact filename written in Step 4. The `&&` guard aborts with a visible `cat` error if the file is not found, preventing OpenCode from receiving an empty prompt.

   Do not use `opencode run "$(cat ...)"` — a `cat` failure inside the subshell produces an empty string that is silently passed to OpenCode as the prompt. The `TASK=` assignment with `&&` is the only permitted form.

6. **Review what OpenCode actually changed**, not what it claims to have changed:

   ```bash
   git diff "$(cat opencode_tasks/baseline.txt)"
   git ls-files --others --exclude-standard | LC_ALL=C sort > opencode_tasks/untracked_after.txt
   comm -13 opencode_tasks/untracked_before.txt opencode_tasks/untracked_after.txt
   ```

   This is the canonical review interface. The output of OpenCode itself is informational; the filesystem diff against the baseline is ground truth. Trust the diff.

7. **Validate with targeted commands** specific to what changed (`pytest path/to/test_file.py`, `npm test -- path/to/test`, `cargo test specific_test`, etc.). Use the project's existing tooling and environment — don't invent commands. Targeted first, broad only if needed.
8. **If the patch is wrong, iterate surgically.** Write a small corrective task that names the exact issue from the diff. Do not re-run the whole task. See `references/examples.md` for a corrective task example.
9. **Confirm the Definition of Done** (next section), then write a short final summary.

## Definition of Done

A task is complete only when:

- the diff against the baseline contains only intended implementation changes
- relevant targeted validation passes, or failures are explicitly explained
- no unrelated files are intentionally included
- the implementation matches the original request
- existing APIs and behavior are preserved unless explicitly changed
- the diff has been reviewed
- no user work has been deleted, reverted, stashed, reset, restored, overwritten, or cleaned

If validation cannot be run (missing dependencies, environment issues), explain why instead of skipping silently. If unrelated files are present, mention them separately and do not touch them.

## OpenCode Task File Format

The task file is the deterministic interface to OpenCode. Treat it as an engineering ticket, not a chat prompt. Write it to `opencode_tasks/YYYYMMDD-<slug>.txt` as described in Step 4.

Minimum required sections:

- **Context** — relevant existing behavior, files already inspected, project patterns to follow
- **Files to modify** — explicit paths, one per line, no globs
- **Files to inspect only** — context files OpenCode should read but not change
- **Required changes** — numbered, each describing a single observable change
- **Implementation guidance** — where to place changes, what helpers to reuse, expected control flow
- **Constraints** — preserve public API, preserve signatures, no unrelated refactors, no new dependencies, no broad formatting, do not stage or commit any changes (required in every task)
- **Tests/validation** — explicit test file path, explicit targeted command
- **Do not** — explicit negative constraints (the temptations OpenCode is likely to indulge)

A typical task fits in 30–60 lines. A corrective task fits in under 15. If a task grows past ~80 lines it probably contains two tasks — split it.

## Minimal Patch Rules

Always instruct OpenCode to:

- modify only necessary files
- keep the patch small
- avoid unrelated refactors
- preserve existing architecture and naming conventions
- avoid formatting unrelated code
- avoid unnecessary renames
- avoid speculative improvements
- avoid new dependencies unless required
- avoid changing public APIs unless required
- do not run git add or git commit
- do not stage any files
- leave all changes unstaged

A clean diff makes review tractable. A noisy diff hides regressions. Broad formatters and "improvements" are forbidden because they pollute the diff, not because they're inherently bad.

## Review Rules

After OpenCode finishes, inspect the diff against the baseline. Never trust the executor's natural-language summary alone — the same model wrote the patch and the summary, so blind spots are shared.

Review for: correctness, completeness, minimality, regressions, edge cases, style consistency, architectural consistency, broken imports/types, accidental edits, unnecessary complexity, modifications outside the declared scope.

If files outside the declared scope are modified or created:

1. Do not revert them automatically.
2. Do not delete them automatically.
3. Do not stash them automatically.
4. Report them to the user.
5. Ask for explicit approval before any cleanup.
6. Continue reviewing the relevant changes in the meantime.

## Iteration Strategy

If the implementation is wrong:

1. Do not rewrite manually.
2. Write a corrective task that points at the exact issue in the diff.
3. Keep the correction surgical: one issue, smallest possible fix.
4. Re-run OpenCode.
5. Re-validate targeted.
6. Review the new diff against the same baseline.

The same `opencode_tasks/baseline.txt` is reusable across multiple iterations — every `git diff "$(cat opencode_tasks/baseline.txt)"` shows the cumulative OpenCode contribution since the baseline was captured. This lets you accept the patch progressively without losing visibility into what's changed.

Avoid broad retries. Avoid asking OpenCode to redo the whole task unless the patch is fundamentally unusable. If you're tempted to retry broadly, the original task was probably too large — split it instead.

## Validation Details

Prefer targeted commands that exercise specifically what changed. The patch surface determines the test surface, not the other way around.

Use the repository's existing scripts and tooling. Prefer the active project environment (current virtualenv, package manager, existing test runner).

Do not blindly trust OpenCode's validation output. After it claims success, rerun the same validation yourself. If validation fails:

1. Inspect the failure.
2. Determine whether it's related to the patch.
3. If related, write a focused corrective task.
4. If unrelated, report it separately. Do not modify unrelated files to make validation pass.

## Failure Handling

If OpenCode fails: inspect the error, reduce task scope, make the next task more explicit, retry incrementally. Avoid broad retries. Do not clean the worktree without user approval.

If OpenCode repeatedly fails on the same task: stop retrying, re-inspect the relevant files yourself, split the task into smaller steps, delegate one small change at a time. Ask the user before switching to direct manual edits — that's a workflow change, not a tactical decision.

## Final Response Style

After completing a task, write a short summary:

- files changed (explicit list)
- behavior implemented (one line)
- tests/checks run and result
- any caveats
- any unrelated files noticed and left untouched

Do not re-narrate the reasoning or the diff. The user can read the diff themselves. The summary confirms scope and validation, nothing more. See `references/examples.md` for an example final response.

## References

Consult these only when needed — they are not preloaded:

- `references/examples.md` — Good vs. bad delegation, corrective task example, final response example
- `references/git-safety.md` — Extended reasoning on the forbidden-commands list and recovery options
