---
name: atomic
description: Atomic development workflow — write one function, test it, commit if green, fix if red. Use this whenever someone wants to implement a feature, add a function, write code incrementally, or follow a TDD dev loop.
argument-hint: "[--no-pr] [--skip-simplify] [task description] or [task id]"
---

## Step 0 — Resolve the task

Parse flags from `$ARGUMENTS` before treating the rest as the task or ID:

- `--no-pr` — skip the PR push step entirely; commit to the current branch only
- `--skip-simplify` — bypass the simplify pass after each commit

Strip parsed flags so the remainder is the task description or numeric ID.

Check if the remainder is a numeric ID or a text prompt:

**If numeric** — look up the task from `.devkit/tasks.db`:
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
- If the task status is `done`, stop and tell the user it's already completed.
- If the task status is `in_progress`, warn the user it was previously started and ask whether to resume or restart.
  - **Resume** — switch to the existing branch and continue from the last completed iteration in the dev loop.
  - **Restart** — ask the user for confirmation, then delete the old feature branch, reset the task status to `pending`, and continue from Step 1 as if starting fresh.
- If found, use `title` + `description` as the resolved task definition for all steps below.

**If text** — use the remainder directly as the task definition.

- If the text references an external issue tracker (Jira, GitHub Issues, Linear), fetch the issue details first.

**After resolving the task (both paths):**

- If the resolved task description contains more than 2 distinct deliverables, touches more than 3 files, or is too vague to implement in a single focused loop, stop and suggest running the **atomize** skill first.
- If the task is ambiguous about language, framework, or key technical decisions, ask the user to clarify before proceeding.

---

## Step 1 — Create a feature branch

Check for `.devkit/batch.json` first:
```bash
cat .devkit/batch.json 2>/dev/null
```
If it exists, read `branch` and `trunk` from it. This means `build` is orchestrating a batch run — the branch is already created and managed by build. Skip all branch creation logic and the PR status check below. Proceed directly to the Implementation section. (You'll also skip the push and PR steps in Finalize — build handles those.)

If the working directory has no source code, ask the user about project setup before proceeding.

Check for uncommitted changes first:
```
git status --short
```
If there are uncommitted changes, stop and ask the user to commit or stash them before continuing.

Check the current branch:
```
git branch --show-current
```

If already on a feature branch (not `main`, `master`, `develop`, or `trunk`), check whether a PR exists for it:
```
gh pr view --json state,title,url 2>/dev/null
```
- **PR is open (not merged)** — tell the user the branch has an open PR, show the title and URL, then ask: "Do you want to add commits to this branch (continuing the open PR), or start a new branch for this task?"
- **PR is merged** — tell the user this branch's PR was already merged. Suggest switching back to trunk.
- **No PR / gh not available** — ask the user whether to continue on the existing branch or create a new one.

If on a trunk branch (`main`, `master`, `develop`, or `trunk`), derive a branch name from the task title:
- Prefix with `feat/`, lowercase, hyphenated, no special chars
- Strip common filler words: "add", "the", "a", "an", "and", "to", "for", "of", "in", "with"
- Keep the most meaningful tokens
- Truncate to **40 characters max** (not counting the `feat/` prefix)

```
git checkout -b feat/<branch-name>
```
If the branch already exists, ask the user whether to switch to it or choose a different name.

Examples of the slug strategy:
- `add POST /users route handler` → `feat/post-users-route-handler` (strip "add")
- `add the user authentication middleware` → `feat/user-authentication-middleware` (strip "add", "the")
- `create database connection pool manager with retry logic` → `feat/database-connection-pool-manager` (truncated at 40)

If the task came from a numeric ID, mark it `in_progress`:
```
uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('in_progress', $ARGUMENTS)); conn.commit(); conn.close()"
```

---

## Implementation — Agent Delegation

The main agent orchestrates, verifies, and enforces quality. Prefer delegating implementation to specialized agents rather than writing code directly.

### Agent roster

| Task type | Agent/Skill | Purpose |
|-----------|------------|---------|
| Backend / general code | **feature-dev:feature-dev** | Codebase understanding + implementation |
| Frontend code | **feature-dev:feature-dev** | Frontend implementation (components, hooks, pages) |
| Frontend UI/UX design | **frontend-design:frontend-design** | UI design, layout, styling, visual decisions |
| Library/package usage | **context7** MCP tools | Fetch up-to-date docs before any agent writes code |
| Infrastructure/deploy | **feature-dev:feature-dev** | Dockerfiles, CI/CD, infrastructure config |
| Code quality | **code-review:code-review** | Review each iteration's output |
| Post-commit cleanup | **simplify** | Review changed code for reuse, quality, efficiency |
| Pre-push review | **pr-review-toolkit:review-pr** | Comprehensive PR review before pushing |

### Agent collaboration

Agents collaborate directly in a pipeline — each agent's output feeds into the next.

**Frontend task** (e.g., build a dashboard page):
1. **context7** → fetch framework/library docs
2. **frontend-design:frontend-design** → produce UI/UX design
3. **feature-dev:feature-dev** → implement components, hooks, pages based on the design
4. **code-review:code-review** → review the implementation

**Backend task** (e.g., add a REST endpoint):
1. **context7** → fetch library/framework docs
2. **feature-dev:feature-dev** → implement route, service logic, data layer
3. **code-review:code-review** → review the implementation

**Worker/job task** (e.g., background email sender, cron job):
1. **context7** → fetch docs for queue/scheduler libraries
2. **feature-dev:feature-dev** → implement the worker logic
3. **code-review:code-review** → review the implementation

**Infrastructure task** (e.g., add Docker setup, CI/CD pipeline):
1. **feature-dev:feature-dev** → create Dockerfiles, compose files, CI config
2. **code-review:code-review** → review the configuration

**Full-stack task** (e.g., user profile page with API):
1. **context7** → fetch docs for both frontend and backend libraries
2. **feature-dev:feature-dev** → implement backend API first
3. **frontend-design:frontend-design** → produce UI/UX design for the frontend
4. **feature-dev:feature-dev** → implement frontend based on the design, wired to the API
5. **code-review:code-review** → review the full stack

**code-review:code-review** is mandatory at the end of every pipeline — do not skip it.

---

## Atomic Dev Loop

One logical unit per iteration, one commit per green test. The main agent orchestrates.

### Progress indicator

At the start of each task run, before any work begins, print a checklist of the steps that will execute based on the task type. This is purely additive output — no behavior changes. Example for a backend task:

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

Mark steps that will be skipped due to flags.

### The loop

Before starting the first iteration, record the current HEAD SHA as the **pre-task SHA**:
```bash
git rev-parse HEAD
```
Store this value — it will be used for rollback if needed.

Track **consecutive gate failure count** (reset to 0 after each successful commit).

Repeat these steps for each logical unit until the task is complete:

1. **Delegate** — trigger the agent pipeline that matches the task type. Pass the task context and any prior fix feedback to the first agent.
2. **Verify** — check the output against all [quality gates](#quality-gates).
3. **Gates pass** — reset consecutive failure count to 0. Commit with a focused message. Then, unless `--skip-simplify` was set, run the **simplify** skill. If simplify produces changes, commit them: `refactor: simplify <what changed>`.
4. **Gates fail** — increment consecutive failure count. Send specific failure details back to the responsible agent for fixes.
   - If consecutive failure count reaches **3**, stop delegating and present this menu to the user:
     ```
     Gate failed 3 times in a row.
     Options:
     (1) continue — send to agent for another fix attempt
     (2) stop — leave the branch as-is for manual resolution
     (3) rollback — reset to pre-task state and mark task pending
     ```
   - **Rollback** — run `git reset --hard <pre-task SHA>`. If the task came from a numeric ID, reset its status to `pending`. Tell the user the branch is clean and the task is back in the queue. Commits from earlier tasks in the same build run are untouched.
   - **Continue** — reset consecutive failure count to 0, delegate again.
   - **Stop** — leave the branch and exit.

### Finalize

Once all loop iterations are done, run these steps in order:

1. **Mark done** — if the task came from a numeric ID:
   ```
   uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('done', <id>)); conn.commit(); conn.close()"
   ```

2. **Append to `.devkit/tasks.md`** — permanent history log. Use `git log origin/<trunk>..HEAD --stat` to scope to this branch only. For numeric tasks: `## task-<id>: <title>`, for text tasks: `## task: <title>`. Create the file if it doesn't exist; always append, never overwrite.
   ```markdown
   ## task-<id>: <title>
   - libraries: <packages added — or "none">
   - files: <files created or modified, comma-separated>
   - decisions: <key design choices made>
   - notes: <what a future session needs to know to continue this task>
   ```

3. **PR review and push** — if `.devkit/batch.json` exists or `--no-pr` was set, skip this step entirely. Otherwise:
   - Run **pr-review-toolkit:review-pr**. If issues found, send back to agents for fixes.
   - Ask user for confirmation, then push: `git push -u origin <branch-name>`
   - Open a PR against trunk summarizing what was implemented.

4. **Update `## Scope` in CLAUDE.md** — if `CLAUDE.md` exists in the project root, rewrite its `## Scope` section as a single, always-current description of what the entire project is and does after this PR. Skip if CLAUDE.md does not exist.

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
   Mark each as `x` (done) or `skip` (with reason).

### Quality gates

The main agent checks every one of these after each agent delivers code. All must pass before committing.

- [ ] Test runner exists — if no test framework is configured, stop and ask the user whether to add one before proceeding.
- [ ] All tests passing
- [ ] Each commit is **100 lines of hand-written implementation code max** (tests, migrations, config, and generated code excluded)
- [ ] If the project has coverage tooling configured, run it and verify coverage meets the project's threshold.
- [ ] No dead code, unused imports, or stubs (run the project's linter if configured)
- [ ] No unrelated changes in the diff
- [ ] Follows existing codebase patterns and conventions
- [ ] No hardcoded secrets or credentials
- [ ] Functions are independently testable (single responsibility)

### Commit message format

```
<verb>: <what the function does>
```

Examples:
- `add: get user by id from database`
- `add: validate email format`
- `fix: handle empty list in paginate`
- `add: create order with default status`

---

## What counts as "one function"

A function is one named, testable unit. It does one thing. A logical unit is a coherent set of functions that can be tested as a group without depending on uncommitted code.

Good — each is independently testable:
- `getUser(id)` — fetches a user by id
- `hashPassword(plain)` — hashes a plain password
- `sendWelcomeEmail(user)` — sends a welcome email

Bad — too broad, split these:
- `setupAuth()` — does too many things
- `handleRequest()` — reads input, validates, queries DB, formats response
- `initApp()` — wires everything together

When related functions must exist together to be testable (e.g., a route handler and its request validation schema), they count as one logical unit and can be committed together.
