# Git Safety Reference

Read this when you need to understand why a specific git command is forbidden, or what to do when unrelated files appear in the worktree. The skill's main file (`SKILL.md`) contains the operational list; this file contains the reasoning and recovery options.

## The Core Rule

The user's worktree contains work that is not yours to manage:

- uncommitted edits in progress
- scratch files and notes
- generated artifacts not yet gitignored
- task files in `opencode_tasks/`
- experiments the user has not decided whether to keep

A planner that touches any of these without explicit permission can destroy hours of work in a single command. The forbidden-commands list exists to make that destruction impossible by default.

The rule is asymmetric on purpose: the cost of one accidental cleanup is permanent data loss; the cost of asking before cleaning is a brief delay.

## Why Each Forbidden Command Is Dangerous

### `git restore <path>` / `git restore .` / `git checkout -- <path>` / `git checkout .`

Replaces working tree contents with index contents. Any unstaged edit in the affected paths is silently discarded. No prompt, no backup, no recovery from reflog (the discarded content was never in any commit).

### `git reset` / `git reset --hard`

`git reset --hard` discards both unstaged and staged changes. `git reset` without `--hard` moves the index pointer and can still drop staged work depending on the form used. Both are appropriate only when the user has explicitly accepted the loss of pending changes.

### `git clean` / `git clean -fd` / `git clean -fdx`

Deletes untracked files. With `-x` it also deletes ignored files, which typically includes build artifacts the user may need. `git clean` is the most common cause of accidental loss of user work because the deleted files were never tracked, so there is nothing to restore from history.

### `git rebase` / `git merge --abort` / `git rebase --abort`

Rewrites or unwinds commit history. Even when not destructive in the strict sense, the new history can confuse the user's mental model of the branch and can lose commits that were not yet pushed. Reserved for the user.

### `git stash push` / `git stash pop` / `git stash apply`

Stashing moves uncommitted work into a stash list. It is reversible in principle, but stashes are easy to lose track of, especially after multiple iterations, and `git stash pop` after an unrelated change can produce conflicts that leave the worktree in a worse state than before. The user manages their own stashes.

### `rm -rf <path>`

The filesystem-level analogue of `git clean`. Unrecoverable. Never used for cleanup unless the user explicitly asks.

## Why `git rev-parse HEAD` Is the Baseline Primitive

The baseline mechanism uses `git rev-parse HEAD`:

- It is purely read-only — it prints the current HEAD SHA and changes nothing in the worktree, the index, or the stash stack.
- The resulting SHA is a plain commit reference. `git diff <SHA>` has deterministic, well-defined behavior: it shows all differences between that commit and the current working tree, whether OpenCode left changes unstaged or committed them.
- `git stash create` was previously considered but produces a multi-parent stash object whose behavior with `git diff` is less predictable, particularly when OpenCode commits its own changes.

This is why the snapshot baseline in the workflow uses `git rev-parse HEAD`, not `git stash create` or `git stash push`.

## Handling Unrelated Files in `git status`

When OpenCode finishes and `git status --short` shows files outside the declared task scope, do not assume OpenCode created them. The user may have been editing in parallel. The files may pre-date the OpenCode run. Even better: use the baseline mechanism — `git diff "$(cat opencode_tasks/baseline.txt)"` shows only what changed *during* the run, automatically excluding pre-existing user work.

For genuinely new files that appeared during the run but weren't part of the task:

1. List them explicitly in your final report.
2. State that they appear newly modified or untracked.
3. Ask the user before reverting, deleting, stashing, overwriting, or otherwise modifying them.
4. Continue reviewing the relevant diff with `git diff "$(cat opencode_tasks/baseline.txt)" -- <path>` for the files that *are* part of the task.

What you must not do:

- run `git restore` to "clean up"
- run `git stash push` to hide them
- run `git clean` to delete untracked files
- include them in the diff you present to the user as the task result

If you are unsure whether a file is related to the task, default to assuming it is unrelated and ask.

## Reviewing Without Cleaning

You can review specific patches without touching the rest of the worktree:

```bash
BASELINE=$(cat opencode_tasks/baseline.txt)
git diff "$BASELINE" -- path/to/changed-file.ext
git diff "$BASELINE" -- path/to/another-file.ext
git diff "$BASELINE" --name-only
```

These are read-only. Use them as the default review path; cleanup is never required in order to review.

## When the User Asks for Cleanup

If the user explicitly authorizes a destructive command, run it — once, with the exact scope they approved. Do not generalize "yes, clean up the OpenCode mess" into "yes, run `git clean -fdx`" without confirming the scope.

Recommended phrasing when confirming:

> I will run `<exact command>` which will <exact effect>, affecting <exact paths>. Confirm?

After the destructive command runs, re-run `git status --short` and report the resulting state.
