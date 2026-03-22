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
  - **Resume** — continue from Step 1 using the existing branch and state.
  - **Restart** — ask the user for confirmation, then delete the old feature branch, reset the task status to `pending`, and continue from Step 2 as if starting fresh.
- If found, use `title` + `description` as the resolved task definition for all steps below.

**If text** — use `$ARGUMENTS` directly as the task definition.

- If the text references an external issue tracker (Jira, GitHub Issues, Linear), fetch the issue details first and use the issue title + description as the task definition.

**After resolving the task (both paths):**

- If the resolved task description contains more than 2 distinct deliverables, touches more than 3 files, or is too vague to implement in a single focused loop, stop and suggest running the **atomize** skill first to break it down before proceeding.
- If the task is ambiguous about language, framework, or key technical decisions (e.g., "add a login endpoint" without specifying the stack), ask the user to clarify before proceeding. Do not assume.

---

## Step 1 — Route by task type

Before writing any code, classify the task and check these routing rules. Apply all that match:

- **If the working directory has no source code** — ask the user about project setup (language, framework, structure) before proceeding.
- **If the task involves frontend UI/UX design** — invoke the **frontend-design:frontend-design** skill for design decisions (layout, styling, visuals), then **feature-dev:feature-dev** for implementation.
- **If the task modifies existing application code** — invoke the **feature-dev:feature-dev** skill to understand the codebase and architecture before implementing.
- **If the task is backend-only** (e.g., REST endpoint, service logic, data layer) — invoke **feature-dev:feature-dev** for implementation.
- **If the task is a worker/job** (e.g., background processor, cron job, queue consumer) — invoke **feature-dev:feature-dev** for implementation.
- **If the task is infrastructure** (e.g., Dockerfiles, CI/CD, deploy config) — invoke **feature-dev:feature-dev** for implementation.
- **If the task is full-stack** (frontend + backend) — invoke **feature-dev:feature-dev** for backend first, then **frontend-design:frontend-design** for UI design, then **feature-dev:feature-dev** for frontend implementation.
- **If the task involves a library or third-party package** — use the **context7** MCP tools (`resolve-library-id` then `query-docs`) to fetch up-to-date documentation before writing any code.

These are mandatory checks, not optional.

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

If already on a feature branch (not `main`, `master`, `develop`, or `trunk`), ask the user whether to continue on it or create a new branch. Do not silently continue on a potentially unrelated branch.

If on a trunk branch (`main`, `master`, `develop`, or `trunk`), derive a branch name from the task title — prefix with `feat/`, lowercase, hyphenated, no special chars — and create it:
```
git checkout -b feat/<branch-name>
```
If the branch already exists, ask the user whether to switch to it or choose a different name.

If the task came from a numeric ID, mark it `in_progress` now that the branch is ready:
```
uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('in_progress', $ARGUMENTS)); conn.commit(); conn.close()"
```

Examples:
- task `add POST /users route handler` → `feat/add-post-users-route-handler`
- task `write get-user-by-id query` → `feat/write-get-user-by-id-query`
- task id `3` with title `create users table migration` → `feat/create-users-table-migration`

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

Agents collaborate directly in a pipeline — each agent's output feeds into the next. The main agent checks quality gates after the pipeline completes.

**Frontend task** (e.g., build a dashboard page):
1. **context7** → fetch framework/library docs
2. **frontend-design:frontend-design** → produce UI/UX design (layout, styling, visuals)
3. **feature-dev:feature-dev** → implement components, hooks, pages based on the design
4. **code-review:code-review** → review the implementation

**Backend task** (e.g., add a REST endpoint):
1. **context7** → fetch library/framework docs
2. **feature-dev:feature-dev** → implement route, service logic, data layer
3. **code-review:code-review** → review the implementation

**Worker/job task** (e.g., background email sender, cron job):
1. **context7** → fetch docs for queue/scheduler libraries
2. **feature-dev:feature-dev** → implement the worker logic, retry handling, scheduling
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

Agents must complete in order — later agents depend on earlier agents' output. For example, **feature-dev:feature-dev** must not start coding until **frontend-design:frontend-design** has finalized the design. Pick the pipeline that matches the task type.

**code-review:code-review** is mandatory at the end of every pipeline — do not skip it. If code-review flags issues, send them back to the responsible agent for fixes before proceeding to quality gates.

---

## Atomic Dev Loop

One logical unit per iteration, one commit per green test. The main agent orchestrates.

### The loop

1. **Delegate** — trigger the agent pipeline based on Step 1 routing. Agents collaborate directly — each agent's output feeds into the next. Pass the task context and any prior fix feedback to the first agent in the pipeline.
2. **Verify** — main agent checks the agent's output against all [quality gates](#quality-gates).
3. **If all gates pass** — commit with a focused message, then you MUST run the **simplify** skill on the changed code (do not skip this step). If simplify produces changes, commit them with message `refactor: simplify <what changed>`.
4. **If any gate fails** — identify which gate(s) failed and send specific failure details back to the responsible subagent for fixes. Max **3 fix cycles** per iteration — if still failing after 3 attempts, stop and ask the user.
5. **Repeat** until the task is complete.
6. **Finalize** — once all iterations are done:
   - If the task came from a numeric ID, mark it `done`:
     ```
     uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('done', <id>)); conn.commit(); conn.close()"
     ```
   - Run **pr-review-toolkit:review-pr** for a comprehensive review. If issues found, send back to agents for fixes.
   - **Ask the user for confirmation** before pushing and opening a PR.
   - If confirmed, push the feature branch: `git push -u origin <branch-name>`
   - Open a pull request against the trunk branch and summarize what was implemented. If the task references an external issue (Jira, GitHub Issues, Linear), include the issue key in the PR title (e.g., `PL-2: add user login endpoint`).
   - Print the completion checklist showing which steps were executed:
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
     ```
     Mark each as `x` (done) or `skip` (with reason). This makes skipped steps visible to the user.

### Quality gates

The main agent checks every one of these after each agent delivers code. All must pass before committing.

- [ ] Test runner exists — if no test framework is configured (no `pytest`, `jest`, `vitest`, `go test`, etc.), stop and ask the user whether to add one before proceeding. Do not silently pass this gate when there are zero tests.
- [ ] All tests passing
- [ ] Each commit is **100 lines of hand-written implementation code max** (tests, migrations, config, and generated code excluded) — if more, split into smaller iterations
- [ ] If the project has coverage tooling configured, run it and verify coverage meets the project's threshold. If no coverage tooling exists, skip this gate.
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
- `setupAuth()` — does too many things (hashing, tokens, middleware)
- `handleRequest()` — reads input, validates, queries DB, formats response
- `initApp()` — wires everything together, not a single unit

When related functions must exist together to be testable (e.g., a route handler and its request validation schema), they count as one logical unit and can be committed together.
