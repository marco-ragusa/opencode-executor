# OpenCode Examples

Read this when you need a concrete example of delegation, a corrective task, or the final summary format. The skill's main file (`SKILL.md`) describes the rules; this file shows them in action.

## Bad vs. Good Delegation

### Bad delegation

```text
Add retry support to the API client.
```

Why it is bad:

- the target file is implicit
- the retry policy is undefined
- the failure conditions to retry are undefined
- there is no constraint on public API or signatures
- the patch surface is unbounded

A small local model will fill the gaps with guesses. The diff will be unpredictable: it might rewrite the fetch wrapper, add an axios dependency, refactor error handling on the side, or all three.

### Good delegation

```text
Modify src/api/client.ts.

Inside the request() function:
- add retry support
- use exponential backoff
- maximum 3 retries
- retry only:
  - network failures
  - HTTP 5xx responses

Do NOT retry:
- HTTP 4xx responses

Preserve:
- existing public API
- existing function signatures
- existing timeout behavior
- existing error types

Implementation guidance:
- keep the current fetch wrapper
- insert the retry loop before final error propagation
- avoid introducing dependencies
- avoid refactoring unrelated logic

Tests:
- add minimal targeted tests in tests/client.test.ts

Do not:
- modify unrelated tests
- reformat unrelated code
- change existing error handling outside the retry path
- run git add or git commit
- stage any files
- leave anything staged or committed
```

Why it is good:

- target file and function are explicit
- retry policy is fully specified (count, backoff, conditions)
- positive and negative conditions are listed
- public API contract is preserved by explicit constraint
- test surface is bounded to a single file
- forbidden behaviors are listed

The more concrete the task, the smaller and more predictable the diff.

## Corrective Task Example

When the first patch is wrong, do not re-run the whole task. Write a small surgical correction that points at the exact issue in the diff.

```text
Update fibonacci.py so generate_fibonacci(n) raises ValueError for negative inputs.

Do not modify unrelated logic.
Do not change the output format.
Do not rename functions.
Do not run git add or git commit.
Do not stage any files.
Leave all changes unstaged.
Keep the patch minimal.
```

This task:

- references a single function
- describes one behavior change
- lists explicit "do not" constraints
- bounds the patch surface

A corrective task should fit in fewer than ~15 lines. If it grows larger, the original delegation was probably too broad and should be split into separate tasks.

## Temporary Task File Workflow

Write the OpenCode prompt to a named file in `opencode_tasks/` before execution. This makes the prompt inspectable, repeatable, and keeps shell history clean.

```bash
mkdir -p opencode_tasks
[ -f opencode_tasks/.gitignore ] || echo "*" > opencode_tasks/.gitignore
cat > "opencode_tasks/$(date +%Y%m%d)-create-fibonacci.txt" <<'EOF'
Create a file named fibonacci.py in the current directory.

Requirements:
- Use Python 3.
- Define generate_fibonacci(n).
- Generate the first 100 Fibonacci numbers.
- Use an iterative approach.
- Store results in a list.
- Print one number per line.
- Include main().
- Include the standard __name__ == "__main__" entrypoint.
- Do not use recursion.
- Do not use external dependencies.
- Do not create additional files.
- Keep the implementation minimal and readable.
- Do not run git add or git commit.
- Do not stage any files.
- Leave all changes unstaged.
EOF

TASK=$(cat "opencode_tasks/YYYYMMDD-create-fibonacci.txt") && opencode run "$TASK"
```

## Full Workflow Example (with baseline)

A complete sequence from baseline capture through final review:

```bash
# 1. Capture baseline
git --version >/dev/null 2>&1 || { echo "ERROR: git is required." >&2; exit 1; }
mkdir -p opencode_tasks
[ -f opencode_tasks/.gitignore ] || echo "*" > opencode_tasks/.gitignore
BASELINE=$(git rev-parse HEAD 2>/dev/null) || { echo "ERROR: repo has no commits — make at least one commit before using this skill." >&2; exit 1; }
echo "$BASELINE" > opencode_tasks/baseline.txt
git ls-files --others --exclude-standard | LC_ALL=C sort > opencode_tasks/untracked_before.txt

# 2. Write the task
cat > "opencode_tasks/$(date +%Y%m%d)-add-retry-client.txt" <<'EOF'
Modify src/api/client.ts.
... [full task here]
EOF

# 3. Run OpenCode — replace YYYYMMDD with today's date from step 2
TASK=$(cat "opencode_tasks/YYYYMMDD-add-retry-client.txt") && opencode run "$TASK"

# 4. Review what actually changed against the baseline
git diff "$(cat opencode_tasks/baseline.txt)"

# 5. Check for new untracked files
git ls-files --others --exclude-standard | LC_ALL=C sort > opencode_tasks/untracked_after.txt
comm -13 opencode_tasks/untracked_before.txt opencode_tasks/untracked_after.txt

# 6. Targeted validation
npm test -- tests/client.test.ts
```

If steps 4–6 reveal issues, write a corrective task at step 2 and repeat steps 3–6. `opencode_tasks/baseline.txt` persists across iterations — every subsequent `git diff "$(cat opencode_tasks/baseline.txt)"` shows the cumulative OpenCode contribution since capture.

## Final Response Example

After completing a task, the final response should be short and structured. No reasoning dump.

```text
Implemented the requested retry behavior.

Changed:
- src/api/client.ts
- tests/client.test.ts

Validation:
- npm test -- tests/client.test.ts passed

Notes:
- I noticed unrelated unstaged files in the worktree and left them untouched.
```

What this includes:

- files changed (explicit list)
- behavior implemented (one line)
- validation command and result
- any unrelated state observed

The user can read the diff themselves. The summary exists to confirm scope and validation, not to re-explain the work.
