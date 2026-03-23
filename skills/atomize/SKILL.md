---
name: atomize
description: Decompose a task prompt into atomic subtasks and save to .devkit/tasks.db. Use whenever a user gives a large or vague task and wants it broken into smallest possible units of work before implementing.
argument-hint: "[task description]"
---

## Step 0 — Validate input

If `$ARGUMENTS` is empty, stop and ask the user to provide a task description. Do not proceed without a task.

If `$ARGUMENTS` references an external issue tracker (Jira key like `PL-2`, GitHub issue `#123`, Linear issue), fetch the issue details first and use the issue title + description as the task definition for all steps below.

---

## Step 1 — Clarify unknowns

Before elaborating, scan the working directory for project files (`pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml`, existing source code) to infer technical decisions already made. Only ask the user about what cannot be inferred from the codebase.

For any remaining unspecified decisions, ask about **all** that are relevant in a single grouped question — do not assume any of them:

- **Language/framework** — e.g. Python/FastAPI, Node/Express, Go, etc.
- **Database** — e.g. SQLite, Postgres, MongoDB, in-memory
- **Auth mechanism** — e.g. JWT, sessions, OAuth (only if auth is relevant to the task)
- **Testing** — should tests be included? If yes, which framework?
- **API style** — REST, GraphQL, RPC (only if an API is being built)
- **Styling approach** — e.g. Tailwind, CSS Modules, styled-components (only if frontend is involved)
- **Component library** — e.g. shadcn/ui, MUI, Radix (only if frontend is involved)
- **State management** — e.g. React Context, Zustand, Redux (only if frontend is involved)

Skip questions that don't apply. If nothing is unclear, proceed directly to Step 2.

---

## Step 2 — Elaborate the task

Restate the original prompt as a detailed description:
- Clarify intent: what is the end goal?
- List assumptions being made
- Define what "done" looks like

Output this as a short paragraph and show it to the user for confirmation before proceeding. If the user has corrections, incorporate them and re-confirm. After two rounds of correction without consensus, ask the user to restate the task from scratch.

Do not proceed to decomposition until the user agrees the elaboration is accurate.

---

## Step 3 — Decompose into atomic tasks

Split the task into atomic units. A function is one named, testable unit that does one thing. When related functions must exist together to be testable (e.g., a route handler and its validation schema), they count as one logical unit.

Each atomic task must:
- Start with a single action verb (add, create, write, configure, update, delete)
- Do exactly one thing — touches one file or one concern
- Have a short `title` (3-8 words) and a one-sentence `description`
- Be assigned an `exec_order` (1, 2, 3...) reflecting dependency order

**Split any task that:**
- Contains "and" — it's two tasks
- Touches more than one file or layer
- Is vague ("set up X", "handle Y") — break it into concrete actions
- Would produce more than 100 lines of implementation code (tests excluded) — if you estimate it will exceed 100 lines, split it

If decomposition yields only one task, tell the user the task is already atomic and suggest running `/devkit:atomic <description>` directly instead of saving to the DB.

**Good atomic tasks (backend):**
- `create users table migration` — adds one schema migration file
- `write get-user-by-id query` — one function, one SQL query
- `add POST /users route handler` — one route, one handler function

**Good atomic tasks (frontend):**
- `create NDA form component` — captures user input for party details
- `add document preview panel` — renders filled template in read-only view
- `add PDF download button` — generates and downloads PDF from template
- `create dashboard layout component` — page shell with sidebar and header

**Bad (split these):**
- `set up database and add user model` — two tasks
- `implement auth` — too vague, split into: create JWT signing function, create token validation middleware, add login route handler
- `build user profile page` — too broad, split into: create profile layout component, add avatar upload form, add user details display

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

---

## Step 5 — Display result

After saving, run this exact query and print the table — do not summarize or reformat:

```
uv run python -c "
import sqlite3, os
db_path = '.devkit/tasks.db'
if not os.path.exists(db_path):
    print('ERROR: .devkit/tasks.db not found. Run atomize first.')
    raise SystemExit(1)
conn = sqlite3.connect(db_path)
rows = conn.execute('SELECT id, exec_order, title, description, status FROM tasks ORDER BY exec_order').fetchall()
conn.close()
id_h, order_h, title_h, status_h = 'ID', 'Order', 'Title', 'Status'
print(f'{id_h:<5} {order_h:<7} {title_h:<45} {status_h}')
print('-' * 70)
for r in rows:
    print(f'{r[0]:<5} {r[1]:<7} {r[2]:<45} {r[4]}')
print()
print(f'Total: {len(rows)} tasks')
print()
print('Use: /devkit:atomic <ID> to execute a task')
"
```

The ID column is what the user passes to `/devkit:atomic`. Always show it.
