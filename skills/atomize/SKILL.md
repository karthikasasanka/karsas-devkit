---
name: atomize
description: Decompose a task prompt into atomic subtasks and save to .devkit/tasks.db. Use whenever a user gives a large or vague task and wants it broken into smallest possible units of work before implementing.
argument-hint: "[task description]"
---

## Step 0 — Validate input

If `$ARGUMENTS` is empty, stop and ask the user to provide a task description. Do not proceed without a task.

If `$ARGUMENTS` references an external issue tracker (Jira key like `PL-2`, GitHub issue `#123`, Linear issue), fetch the full issue details — title, description, and any acceptance criteria. Use this fetched text as the task definition for all steps below. The full text (not the issue key) becomes `parent_prompt` in Step 4, so every downstream agent has the complete requirement rather than just a lookup reference. Verify the fetched text is complete before proceeding.

---

## Step 1 — Clarify unknowns

Before elaborating, do two things:

**Read project context** — check for `CLAUDE.md` in the project root. Note any resources it documents: template directories, catalogs, schemas, APIs, or conventions. Also scan for files directly relevant to the task — if the task involves documents, look for `templates/`; if it involves data, look for schema files or seed data. Record what you find; these discovered resources become constraints you weave into task descriptions in Step 3.

**Infer technical decisions** — scan for project files (`pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml`, existing source code) to understand the stack already in use. Only ask the user about what cannot be inferred.

For any remaining unspecified decisions that affect how the task breaks down, ask about **all** that are relevant in a single grouped question — do not assume any of them:

- **Language/framework** — e.g. Python/FastAPI, Node/Express, Go, etc.
- **Database** — e.g. SQLite, Postgres, MongoDB, in-memory
- **Auth mechanism** — e.g. JWT, sessions, OAuth (only if auth is relevant to the task)
- **Testing** — should tests be included? If yes, which framework?
- **API style** — REST, GraphQL, RPC (only if an API is being built)

Only ask about decisions that change how the task decomposes. Implementation details like styling approach, component libraries, or state management belong in the execution phase — skip them here.

Skip questions that don't apply. If nothing is unclear, proceed directly to Step 2.

---

## Step 2 — Elaborate the task

Restate the original prompt as a detailed description:
- Clarify intent: what is the end goal?
- List assumptions being made
- Define what "done" looks like

Output this as a short paragraph and show it to the user for confirmation before proceeding. If the user has corrections, incorporate them and re-confirm. If after a few rounds alignment still isn't converging, suggest the user restate the task in their own words — the original framing may be the problem.

Do not proceed to decomposition until the user agrees the elaboration is accurate.

---

## Step 3 — Decompose into atomic tasks

Split the task into atomic units. A function is one named, testable unit that does one thing. When related functions must exist together to be testable (e.g., a route handler and its validation schema), they count as one logical unit.

**No self-referential test tasks.** The `/devkit:atomic` skill already writes and runs tests as part of its own dev loop. Do not create tasks like "write tests for X" where X is another task in the same list — that's redundant work. Before saving, scan the list and remove any task that exists only to test an implementation task within the same decomposition.

Exception: if the user's prompt is explicitly about testing (e.g., "write tests for the auth module", "add test coverage to the payments service"), then test tasks are the deliverable. In that case, decompose them the same way — one task per concern, one file or one module per task — by reading the existing codebase to understand what needs covering.

Each implementation task must:
- Start with a single action verb (add, create, write, configure, update, delete)
- Do exactly one thing — touches one file or one concern
- Have a short `title` (3-8 words) and a one-sentence `description` that says **what gets created and where**, naming any existing files, templates, schemas, or APIs from Step 1 that the implementation should build on — enough for an agent to start without re-reading the original prompt or guessing project conventions
- Be assigned an `exec_order` (1, 2, 3...) reflecting dependency order

**Split any task that:**
- Contains "and" — it's two tasks
- Touches more than one file or layer
- Is vague ("set up X", "handle Y") — break it into concrete actions
- Would produce more than 100 lines of implementation code — if you estimate it will exceed 100 lines, split it

If decomposition yields only one task, tell the user the task is already atomic. Offer two options: run `/devkit:atomic <description>` directly, or save it to the DB if they want it tracked.

If decomposition yields more than 20 tasks, the original prompt is likely too broad for a single decomposition. Suggest splitting it into 2-3 high-level milestones and running atomize on each one separately.

**Right granularity for frontend tasks** — frontend tasks should be component-level, not function-level. A form with three fields is one task ("create user settings form"), not three tasks (one per field). Think about what a developer would build in one sitting as a single reviewable unit: a component, a page section, an API call hook, a layout. Don't decompose inside a component unless it's genuinely large (would exceed 100 lines).

**Good atomic tasks (backend):**
- `create users table migration` — adds one schema migration file
- `write get-user-by-id query` — one function, one SQL query
- `add POST /users route handler` — one route, one handler function

**Good atomic tasks (frontend):**
- `create user settings form` — form with display name, email, and avatar fields
- `add document preview panel` — renders filled template in read-only view
- `add PDF download button` — generates and downloads PDF from template
- `create dashboard layout component` — page shell with sidebar and header

**Bad (split these):**
- `set up database and add user model` — two tasks
- `implement auth` — too vague, split into: create JWT signing function, create token validation middleware, add login route handler
- `build user profile page` — too broad, split into: create profile layout component, add avatar upload form, add user details display

**Bad (too granular for frontend):**
- `create display name input field` — this belongs inside a settings form component, not its own task
- `write handleSubmit function` — internal implementation detail, not an atomic deliverable

**Completeness check** — before saving, list each requirement from the Step 2 elaboration and show which task covers it. A requirement with no covering task is a gap; add a task for it before saving. Present this as a coverage table alongside the task list so gaps are obvious:

| Requirement | Covered by |
|---|---|
| user can download completed document | #4 add PDF download button |
| form values substitute into template | #2 create NDA document component |

Get the user's confirmation before saving.

---

## Step 4 — Save to SQLite

Save all tasks to `.devkit/tasks.db` using this Python script via `uv run python`:

```python
import sqlite3, os

# Populate this list with decomposed tasks from Step 3.
# Each task dict must have: title, description, exec_order.
tasks = [
    # {"title": "create users table migration", "description": "adds one schema migration file", "exec_order": 1},
]

# Set this to the original user prompt text.
parent_prompt = ""

if not parent_prompt:
    raise SystemExit("ERROR: parent_prompt must be set to the original user prompt")
if not tasks:
    raise SystemExit("ERROR: tasks list is empty — nothing to save")

db_dir = ".devkit"
os.makedirs(db_dir, exist_ok=True)
conn = sqlite3.connect(os.path.join(db_dir, "tasks.db"))
conn.execute("""
CREATE TABLE IF NOT EXISTS tasks (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    parent_prompt TEXT NOT NULL,
    title TEXT NOT NULL,
    description TEXT NOT NULL,
    status TEXT NOT NULL DEFAULT 'pending',
    exec_order INTEGER NOT NULL,
    created_at TEXT NOT NULL DEFAULT (datetime('now'))
)
""")
conn.commit()

# Check for existing tasks with the same parent_prompt
existing = conn.execute(
    "SELECT COUNT(*) FROM tasks WHERE parent_prompt = ?", (parent_prompt,)
).fetchone()[0]
if existing > 0:
    active = conn.execute(
        "SELECT COUNT(*) FROM tasks WHERE parent_prompt = ? AND status IN ('in_progress', 'done')",
        (parent_prompt,)
    ).fetchone()[0]
    if active > 0:
        print(f"WARNING: {active} tasks are in_progress or done — clearing will lose progress.")
    print(f"Clearing {existing} existing tasks for this prompt before re-inserting.")
    conn.execute("DELETE FROM tasks WHERE parent_prompt = ?", (parent_prompt,))
    conn.commit()

for t in tasks:
    conn.execute(
        "INSERT INTO tasks (parent_prompt, title, description, exec_order) VALUES (?, ?, ?, ?)",
        (parent_prompt, t["title"], t["description"], t["exec_order"])
    )
conn.commit()
conn.close()
print(f"Saved {len(tasks)} tasks to .devkit/tasks.db")
```

Write this as `.devkit/save_tasks.py`, populate the `tasks` list and `parent_prompt`, run it with `uv run python .devkit/save_tasks.py`, then delete it. If the script exits with a non-zero status, stop and show the error to the user. Do not proceed to Step 5. All automation files must stay inside `.devkit/` — never write temp scripts to the project root.

After the script completes successfully, update `.devkit/tasks.md`:

- **If tasks.md does not exist** — create it with a header section for this decomposition.
- **If tasks.md already exists** — append a new section below the existing content (do not overwrite — prior completion notes from earlier tasks must be preserved).

The header section format (use actual IDs from the DB, not the list index — query after saving):

```
# <first 80 chars of parent_prompt>

| ID | Order | Title | Status |
|----|-------|-------|--------|
| 1  | 1     | create users table migration | pending |
...
```

Fetch the actual IDs by querying the DB immediately after saving (use the same parent_prompt filter as Step 5). This file is the persistent context log — each completed task appends notes below these headers.

---

## Step 5 — Display result

After saving, run this exact query and print the table — do not summarize or reformat. Pass the same `parent_prompt` value used in Step 4 so only the current decomposition is shown:

```
uv run python -c "
import sqlite3, os, sys
db_path = '.devkit/tasks.db'
if not os.path.exists(db_path):
    print('ERROR: .devkit/tasks.db not found. Run atomize first.')
    raise SystemExit(1)
parent_prompt = sys.argv[1]
conn = sqlite3.connect(db_path)
rows = conn.execute('SELECT id, exec_order, title, status FROM tasks WHERE parent_prompt = ? ORDER BY exec_order', (parent_prompt,)).fetchall()
conn.close()
print(f'{"ID":<5} {"Order":<7} {"Title":<45} {"Status"}')
print('-' * 70)
for r in rows:
    print(f'{r[0]:<5} {r[1]:<7} {r[2]:<45} {r[3]}')
print()
print(f'Total: {len(rows)} tasks')
print()
print('Use: /devkit:atomic <ID> to execute a task')
" "<the actual parent_prompt value from Step 4>"
```

The ID column is what the user passes to `/devkit:atomic`. The atomic skill looks up the task by ID, reads its title and description, and uses them as the task definition for its dev loop.
