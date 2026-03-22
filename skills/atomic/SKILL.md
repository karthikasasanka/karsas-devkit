---
name: atomic
description: Atomic development workflow — write one function, test it, commit if green, fix if red. Use this whenever someone wants to implement a feature, add a function, write code incrementally, or follow a TDD dev loop.
argument-hint: "[task description] or [task id]"
---

## Step 0 — Resolve the task

Check if `$ARGUMENTS` is a numeric ID or a text prompt:

**If numeric** — look up the task from `.devkit/tasks.db`:
```
uv run python -c "
import sqlite3
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
- If the task status is `in_progress`, warn the user it was previously started and ask whether to resume or restart before continuing.
- If found, use `title` + `description` as the task definition for all steps below.
- Mark it `in_progress` immediately:
  ```
  uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('in_progress', $ARGUMENTS)); conn.commit(); conn.close()"
  ```

**If text** — use `$ARGUMENTS` directly as the task definition.

- If the text references an external issue tracker (Jira, GitHub Issues, Linear), fetch the issue details first and use the issue title + description as the task definition.
- If the description contains more than 2 distinct deliverables, touches more than 3 files, or is too vague to implement in a single focused loop, stop and suggest running the **atomize** skill first to break it down before proceeding.

---

Task: $ARGUMENTS

---

## Step 1 — Route by task type

Before writing any code, check these routing rules:

- **If the task involves frontend UI/UX design** → invoke the **frontend-design** skill for design decisions (layout, styling, visuals), then **feature-dev** for implementation.
- **If code already exists** → invoke the **feature-dev** skill to understand the codebase and architecture before implementing.
- **If the task involves a library or third-party package** → use the **context7** MCP tool to fetch up-to-date documentation before writing any code.
These are mandatory checks, not optional. Apply all that match.

---

## Step 2 — Create a feature branch

Check for uncommitted changes first:
```
git status --short
```
If there are uncommitted changes, stop and ask the user to commit or stash them before continuing. Do not carry unintended changes onto a new branch.

Check the current branch:
```
git branch --show-current
```

If already on a feature branch (not `main` or `master`), ask the user whether to continue on it or create a new branch from `main`/`master`. Do not silently continue on a potentially unrelated branch.

If on `main` or `master`, derive a branch name from the task title — lowercase, hyphenated, no special chars — and create it:
```
git checkout -b <branch-name>
```

Examples:
- task `add POST /users route handler` → `feat/add-post-users-route-handler`
- task `write get-user-by-id query` → `feat/write-get-user-by-id-query`
- task id `3` with title `create users table migration` → `feat/create-users-table-migration`

---

## Implementation — Agent Delegation

The main agent never writes code directly. All implementation is delegated to specialized agents. The main agent orchestrates, verifies, and enforces quality.

### Agent roster

| Task type | Agent/Skill | Purpose |
|-----------|------------|---------|
| Backend / general code | **feature-dev** | Codebase understanding + implementation |
| Frontend code | **feature-dev** | Frontend implementation (components, hooks, pages) |
| Frontend UI/UX design | **frontend-design** | UI design, layout, styling, visual decisions |
| Library/package usage | **context7** MCP tool | Fetch up-to-date docs before any agent writes code |
| Code quality | **code-review** | Review each iteration's output |
| Infrastructure/deploy | **devops agent** | Dockerfiles, CI/CD, infrastructure |
| Post-commit cleanup | **simplify** | Review changed code for reuse, quality, efficiency |
| Pre-push review | **pr-review-toolkit** | Comprehensive PR review before pushing |

### Agent collaboration

Agents collaborate directly in a pipeline — each agent's output feeds into the next. The main agent does not mediate between agents; it only checks quality gates after the pipeline completes.

**Frontend task** (e.g., build a dashboard page):
1. **context7** → fetch framework/library docs
2. **frontend-design** → produce UI/UX design (layout, styling, visuals)
3. **feature-dev** → implement components, hooks, pages based on the design
4. **code-review** → review the implementation

**Backend task** (e.g., add a REST endpoint):
1. **context7** → fetch library/framework docs
2. **feature-dev** → implement route, service logic, data layer
3. **code-review** → review the implementation

**Worker/job task** (e.g., background email sender, cron job):
1. **context7** → fetch docs for queue/scheduler libraries
2. **feature-dev** → implement the worker logic, retry handling, scheduling
3. **code-review** → review the implementation

**Infrastructure task** (e.g., add Docker setup, CI/CD pipeline):
1. **devops agent** → create Dockerfiles, compose files, CI config
2. **code-review** → review the configuration

**Full-stack task** (e.g., user profile page with API):
1. **context7** → fetch docs for both frontend and backend libraries
2. **feature-dev** → implement backend API first
3. **frontend-design** → produce UI/UX design for the frontend
4. **feature-dev** → implement frontend based on the design, wired to the API
5. **code-review** → review the full stack

Agents must complete in order — later agents depend on earlier agents' output. For example, **feature-dev** must not start coding until **frontend-design** has finalized the design. Pick the pipeline that matches the task type.

---

## Atomic Dev Loop

One function per iteration, one commit per green test. The main agent orchestrates — it does not implement.

### The loop

1. **Delegate** — trigger the agent pipeline based on Step 1 routing. Agents collaborate directly — each agent's output feeds into the next. Pass the task context and any prior fix feedback to the first agent in the pipeline.
2. **Verify** — main agent checks the agent's output against all [quality gates](#quality-gates).
3. **If all gates pass** — commit with a focused message, then run the **simplify** skill on the changed code. If the task came from a numeric ID, also mark it done:
   ```
   uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('done', <id>)); conn.commit(); conn.close()"
   ```
4. **If any gate fails** — identify which gate(s) failed and send specific failure details back to the responsible subagent for fixes. Max **3 fix cycles** per iteration — if still failing after 3 attempts, stop and ask the user.
5. **Repeat** until the task is complete.
6. **Finalize** — once all iterations are done:
   - Run **pr-review-toolkit** for a comprehensive review. If issues found, send back to agents for fixes.
   - Push the feature branch: `git push -u origin <branch-name>`
   - Open a pull request against `main`/`master` and summarize what was implemented. If the task references an external issue (Jira, GitHub Issues, Linear), include the issue key in the PR title (e.g., `PL-2: add user login endpoint`).
   - If the task came from a numeric ID, confirm it is marked `done` in `.devkit/tasks.db`.

### Quality gates

The main agent checks every one of these after each agent delivers code. All must pass before committing.

- [ ] All tests passing
- [ ] Each commit is **100 lines of changed code max** — if more, split into smaller iterations
- [ ] Code coverage is **95% minimum**
- [ ] No dead code, unused imports, or stubs
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

