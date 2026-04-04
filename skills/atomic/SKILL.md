---
name: atomic
description: Atomic development workflow — write one function, test it, commit if green, fix if red. Use this whenever someone wants to implement a feature, add a function, write code incrementally, or follow a TDD dev loop.
argument-hint: "[--no-pr] [--skip-simplify] [task description] or [task id]"
---

## Step 0 — Resolve the task

Parse flags from `$ARGUMENTS` first:

- `--no-pr` — skip the PR push step; commit to current branch only
- `--skip-simplify` — bypass the simplify pass after each commit

Strip parsed flags; the remainder is the task description or numeric ID.

**If numeric** — look up from `.devkit/tasks.db`:
```
uv run python -c "
import sqlite3, os
if not os.path.exists('.devkit/tasks.db'):
    raise SystemExit('.devkit/tasks.db not found — run the atomize skill first to create tasks')
conn = sqlite3.connect('.devkit/tasks.db')
row = conn.execute('SELECT id, title, description, status FROM tasks WHERE id = ?', ($ARGUMENTS,)).fetchone()
conn.close()
if not row: raise SystemExit('Task $ARGUMENTS not found in .devkit/tasks.db')
print(f'id={row[0]} | status={row[3]}')
print(f'title: {row[1]}')
print(f'description: {row[2]}')
"
```
- `done` → stop, tell user it's already completed.
- `in_progress` → warn, ask to resume (switch to existing branch, continue from last iteration) or restart (confirm, delete branch, reset to `pending`, start fresh from Step 1).
- Found → use `title` + `description` as the task definition.

**If text** — use the remainder directly. If it references an external issue tracker (Jira, GitHub Issues, Linear), fetch the issue details first.

**After resolving (both paths):**
- If the task has more than 2 distinct deliverables, touches more than 3 files, or is too vague for a single focused loop — stop and suggest running **atomize** first.
- If language, framework, or key technical decisions are ambiguous — ask before proceeding.

---

## Step 1 — Create a feature branch

Check for `.devkit/batch.json` first:
```bash
cat .devkit/batch.json 2>/dev/null
```
If it exists, read `branch` and `trunk` from it — build is orchestrating a batch run. Skip all branch/PR logic and proceed directly to Implementation. (Also skip push/PR in Finalize — build handles those.)

If the working directory has no source code, ask about project setup before proceeding.

Check for uncommitted changes:
```
git status --short
```
If any exist, stop and ask the user to commit or stash before continuing.

Check current branch:
```
git branch --show-current
```

If already on a feature branch (not `main`, `master`, `develop`, `trunk`):
```
gh pr view --json state,title,url 2>/dev/null
```
- **PR open** — show title/URL, ask: add commits to this branch or start a new one?
- **PR merged** — tell user; suggest switching back to trunk.
- **No PR / gh unavailable** — ask whether to continue on existing branch or create a new one.

If on trunk, derive a branch name from the task title:
- Prefix `feat/`, lowercase, hyphenated, no special chars
- Strip filler words: "add", "the", "a", "an", "and", "to", "for", "of", "in", "with"
- Truncate to **40 characters max** (not counting `feat/`)

```
git checkout -b feat/<branch-name>
```
If branch already exists, ask to switch to it or choose a different name.

Examples:
- `add POST /users route handler` → `feat/post-users-route-handler`
- `add the user authentication middleware` → `feat/user-authentication-middleware`
- `create database connection pool manager with retry logic` → `feat/database-connection-pool-manager`

If the task came from a numeric ID, mark it `in_progress`:
```
uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('in_progress', $ARGUMENTS)); conn.commit(); conn.close()"
```

---

## Implementation — Agent Delegation

The main agent orchestrates, verifies, and enforces quality. Delegate to specialized agents for non-trivial tasks; implement inline for simple ones (agents add latency not justified for small changes).

**Implement inline** (no feature-dev, no code-review) when the task is a refactor, rename, swap, or removal — or changes ≤ 3 files and ≤ 50 lines with existing tests. Read the files, apply the change, run tests.

**Delegate to agents** for new functionality, business logic, or multiple interacting components.

### Agent roster

| Task type | Agent/Skill | Purpose |
|-----------|------------|---------|
| Backend / general code | **feature-dev:feature-dev** | Codebase understanding + implementation |
| Frontend code | **feature-dev:feature-dev** | Frontend implementation (components, hooks, pages) |
| Frontend UI/UX design | **frontend-design:frontend-design** | UI design, layout, styling, visual decisions |
| Library/package usage | **context7** MCP tools | Fetch up-to-date docs before any agent writes code |
| Infrastructure/deploy | **feature-dev:feature-dev** | Dockerfiles, CI/CD, infrastructure config |
| Pre-commit cleanup | **simplify** | Reduce redundancy, remove dead code, tighten style |
| Pre-commit review | **code-review:code-review** | Review after simplify, before committing |

If you already resolved a context7 library ID earlier in this build session, reuse it — skip the resolve call.

### Agent pipelines by task type

| Task type | Pipeline |
|-----------|----------|
| Backend | context7 docs → feature-dev |
| Frontend | context7 docs → frontend-design → feature-dev |
| Full-stack | context7 docs → feature-dev (API) → frontend-design → feature-dev (UI) |
| Worker/job | context7 docs → feature-dev |
| Infrastructure | feature-dev only |

Review happens after simplify in the dev loop, not inside the pipeline.

---

## Atomic Dev Loop

One logical unit per iteration, one commit per green test. The main agent orchestrates.

### Progress indicator

Before any work begins, print a checklist of steps that will execute (mark skipped steps due to flags):

```
Steps for this task:
[ ] fetch context7 docs
[ ] delegate to feature-dev
[ ] code-review
[ ] run quality gates
[ ] commit
[ ] simplify  (skipped: --skip-simplify)
[ ] push + open PR  (skipped: --no-pr)
```

### The loop

Record the current HEAD as **pre-task SHA** before starting:
```bash
git rev-parse HEAD
```

Track **consecutive gate failure count** (reset to 0 after each successful commit).

Repeat for each logical unit until the task is complete:

1. **Delegate** — trigger the agent pipeline for the task type. Pass task context and any prior fix feedback.
2. **Verify** — check output against all [quality gates](#quality-gates).
3. **Gates pass** — reset failure count. Before committing, clean up staged changes:
   - `git diff --stat | tail -1` to count lines changed
   - **≤ 50 lines**: review inline — scan diff yourself, apply fixes directly.
   - **51–200 lines**: run **simplify**, then **code-review:code-review**.
   - **> 200 lines**: run **simplify** (full pass), then **code-review:code-review**.
   Re-run tests after any fixes. Commit everything in one clean commit.
4. **Gates fail** — increment failure count. Send specific failure details back to the agent.
   - At **3 consecutive failures**, stop and present:
     ```
     Gate failed 3 times in a row.
     Options:
     (1) continue — send to agent for another fix attempt
     (2) stop — leave the branch as-is for manual resolution
     (3) rollback — reset to pre-task state and mark task pending
     ```
   - **Rollback**: `git reset --hard <pre-task SHA>`. Reset task status to `pending` if from a numeric ID.
   - **Continue**: reset failure count, delegate again.
   - **Stop**: leave the branch and exit.

### Finalize

Once all loop iterations are done:

1. **Mark done** — if from a numeric ID:
   ```
   uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('done', <id>)); conn.commit(); conn.close()"
   ```

2. **Append to `.devkit/tasks.md`** — use `git log origin/<trunk>..HEAD --stat` to scope to this branch. Create if missing; always append. Format:
   ```markdown
   ## task-<id>: <title>
   - libraries: <packages added — or "none">
   - files: <files created or modified, comma-separated>
   - decisions: <key design choices made>
   - notes: <what a future session needs to know to continue this task>
   ```

3. **Push** — skip if `.devkit/batch.json` exists or `--no-pr` was set. Otherwise confirm with user, then:
   - `git push -u origin <branch-name>`
   - Open a PR against trunk summarizing what was implemented.

4. **Update `## Scope` in CLAUDE.md** — if `CLAUDE.md` exists in the project root, rewrite its `## Scope` section to reflect the current state of the project after this PR. Skip if CLAUDE.md does not exist.

5. **Print completion checklist**:
   ```
   Atomic workflow complete:
   [x/skip] context7 docs fetched
   [x/skip] feature-dev delegated
   [x/skip] frontend-design delegated
   [x/skip] code-review ran
   [x/skip] simplify ran
   [x/skip] pr-review-toolkit ran
   [x/skip] tests passing
   [x/skip] pushed + PR opened
   [x/skip] CLAUDE.md updated
   ```

### Quality gates

All must pass before committing:

- [ ] Test runner exists — if not configured, ask whether to add one before proceeding
- [ ] All tests passing
- [ ] Each commit is **≤ 100 lines of hand-written implementation code** (tests, migrations, config, generated code excluded)
- [ ] Coverage meets project threshold (if coverage tooling is configured)
- [ ] No dead code, unused imports, or stubs (run linter if configured)
- [ ] No unrelated changes in the diff
- [ ] Follows existing codebase patterns and conventions
- [ ] No hardcoded secrets or credentials
- [ ] Functions are independently testable (single responsibility)

### Commit message format

```
<verb>: <what the function does>
```

Examples: `add: get user by id from database` / `fix: handle empty list in paginate`

---

## What counts as "one function"

A function is one named, testable unit that does one thing. Related functions that must exist together to be testable (e.g., a route handler and its validation schema) count as one logical unit.

**Good** (independently testable): `getUser(id)`, `hashPassword(plain)`, `sendWelcomeEmail(user)`

**Bad** (split these): `setupAuth()`, `handleRequest()`, `initApp()` — each does too many things
