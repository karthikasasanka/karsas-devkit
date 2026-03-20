---
name: atomic
description: Atomic development workflow — scaffold structure or write one function, test it, commit if green, fix if red. Use this whenever someone wants to implement a feature, add a function, write code incrementally, scaffold a new project, or follow a TDD dev loop.
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
- If found, use `title` + `description` as the task definition for all steps below.
- Mark it `in_progress` immediately:
  ```
  uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('in_progress', $ARGUMENTS)); conn.commit(); conn.close()"
  ```

**If text** — use `$ARGUMENTS` directly as the task definition.

---

Task: $ARGUMENTS

---

## Step 1 — Create a feature branch

Check the current branch:
```
git branch --show-current
```

If already on a feature branch (not `main` or `master`), continue on it.

If on `main` or `master`, derive a branch name from the task title — lowercase, hyphenated, no special chars — and create it:
```
git checkout -b <branch-name>
```

Examples:
- task `add POST /users route handler` → `feat/add-post-users-route-handler`
- task `write get-user-by-id query` → `feat/write-get-user-by-id-query`
- task id `3` with title `create users table migration` → `feat/create-users-table-migration`

---

## Step 2 — Check for existing code

Scan the current directory for source files. Two paths:

- **No code exists** → go to [Bootstrap](#bootstrap)
- **Code exists** → go to [Atomic Dev Loop](#atomic-dev-loop)

---

## Bootstrap

Generate a `PROJECT_STRUCTURE.md` that shows the minimal base structure for the framework or library mentioned in the task.

### What to include

- **Tree layout** — folders and files only, no implementation
- **File responsibilities** — one line per file describing its sole purpose
- **Dependencies** — minimal list for the project's package manager file (e.g., `pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml`, `pom.xml`)
- **One concrete starter example** — described in plain text: what the entry point is, what it does, what a minimal first route/command/handler looks like

### Rules

- Only include files that are necessary to start. No `utils`, no `helpers`, no base classes unless the framework requires them.
- Do not write code in the structure doc. Describe intent in plain text.
- Do not create the actual files. Write the markdown only.
- Stop and ask the user to confirm the structure before doing anything else.
- After the user confirms and files are created: run `git init .`, create `.gitignore` and `.dockerignore`, then commit immediately with: `init: base project structure`
- Never assume any runtime, compiler, or toolchain is installed on the user's machine. Always use Docker to build and run. Include a `Dockerfile` in the project structure.

### What the starter example should describe (not code)

Instead of writing actual code, describe:
- The entry point file and its role
- The first working endpoint, command, or handler — what it accepts and what it returns
- How to run it locally

Example description for a REST API:
> Entry point: `main` file in the project root. Registers a single health route at `GET /health` that returns `{ status: ok }`. Run with the framework's dev server command.

Example description for a CLI tool:
> Entry point: `main` file. Registers one command `hello` that prints a greeting to stdout. Run with `<runtime> run hello`.

### Antipatterns to avoid in bootstrap

| Antipattern | Why it's wrong |
|---|---|
| Adding `utils` or `helpers` upfront | You don't know what's shared yet |
| Creating a `base` or `core` folder | Premature abstraction |
| Adding auth, logging, or middleware before the first working route | Build one thing first |
| More than one starter example | Increases ambiguity, not clarity |
| Generating actual files without asking | User may want to adjust the structure first |
| Writing code instead of describing it | Structure doc should be readable, not runnable |
| Omitting a Dockerfile | Never assume the user has runtimes installed locally — Docker is always required |
| Running tests without Docker | Use containers; do not rely on locally installed tools |

---

## Minimal Working Code

"Minimal working" means: the function passes its tests and handles only the cases stated in the task. Nothing more.

### What to include

- The logic needed to satisfy the task
- Imports that are actually used
- Error handling only for cases that genuinely can't be handled (e.g., missing required input)

### What to omit unless explicitly asked

| Omit | Unless |
|---|---|
| Inline documentation / docstrings / JSDoc | User asks for documentation |
| Type annotations / type hints | Project already uses them consistently |
| Logging statements | User asks for observability |
| Exception catching / try-catch | The function can genuinely fail in a recoverable way |
| Config or environment variable loading | The function needs external config to work |
| Abstract base classes or interfaces | There are already 2+ concrete implementations |
| Module re-exports | The module is a published package |
| Inline comments | The logic is genuinely non-obvious |

### Before writing, read first

If code already exists:
1. Read 1-2 similar functions in the codebase to understand the existing style
2. Match it — same naming convention, same error handling pattern, same file organization
3. Do not introduce a new pattern if one already exists

### Definition of done

A function is done when:
- [ ] Tests are green
- [ ] It is committed with a focused message
- [ ] Nothing extra was added (no unused imports, no dead code, no stubs)

If any of these are not true, you are not done.

---

## Atomic Dev Loop

Follow this loop strictly. One function per iteration, one commit per green test.

### The loop

1. **Write one function** — the smallest unit that satisfies the task. No helpers, no extras alongside it.
2. **Write tests** — cover the happy path and one meaningful edge case. Keep tests short.
3. **Run the tests** — execute them immediately using Docker. Do not assume the user has any runtime or toolchain installed locally. Use `docker build` and `docker run` (or `docker compose`) to run tests in a container.
4. **If green** — commit with a single focused message. If the task came from a numeric ID, also mark it done:
   ```
   uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('done', <id>)); conn.commit(); conn.close()"
   ```
5. **If red** — diagnose the failure, fix the function or the test (whichever is wrong), return to step 3.

### Commit message format

```
<verb>: <what the function does>
```

Examples:
- `add: get user by id from database`
- `add: validate email format`
- `fix: handle empty list in paginate`
- `add: create order with default status`

### What counts as "one function"

A function is one named, testable unit. It does one thing.

Good — each is independently testable:
- `getUser(id)` — fetches a user by id
- `hashPassword(plain)` — hashes a plain password
- `sendWelcomeEmail(user)` — sends a welcome email

Bad — too broad, split these:
- A create-user function that hashes a password, inserts to DB, and sends email — that's three functions
- A process-order function that validates, calculates, and saves — split it

### What good tests look like

Tests should be:
- One assertion per test case
- Named to describe the scenario, not the implementation
- Independent — no shared mutable state between tests

Good:
- `test that getUser returns the correct user for a valid id`
- `test that getUser throws a not-found error for an unknown id`

Bad:
- A single test that calls create, then get, then checks email was sent — that's three tests
- A test named `test user stuff` or `test it works`
- An assertion that only checks the result is not null — that's too weak

### Antipatterns to avoid in the loop

| Antipattern | Why it's wrong |
|---|---|
| Writing two functions before testing | You lose isolation — hard to tell which one broke |
| Skipping tests because "it's simple" | Simple functions break too; tests also document intent |
| Fixing a failing test by weakening the assertion | The test is telling you something. Listen to it. |
| Committing with all tests in one go | Each commit should be independently meaningful |
| Adding error handling for cases that can't happen | Defensive code that's never tested is noise |
| Refactoring while the loop is running | Finish the function first, refactor in a separate commit |
| Writing a helper before it's needed | Write it when the second function needs it, not the first |

### When tests keep failing

1. Re-read the function — is it actually doing what you think?
2. Add a temporary debug output mid-function to verify state
3. Check if the test setup is wrong (wrong fixture, wrong mock, wrong input)
4. Do not retry the same fix more than twice — step back and rethink the approach
5. Do not move on with a skipped or commented-out test

---

## When to stop and ask the user

Do not silently assume. Stop and ask when:

- The task is ambiguous about what the function should return
- It's unclear whether to throw an error or return a default value
- The task would require touching more than one existing file
- A dependency is not already in the project and you're about to add it
- The happy path requires a decision that has tradeoffs (e.g., sync vs async, in-memory vs persisted)

One short question is better than a wrong implementation.
