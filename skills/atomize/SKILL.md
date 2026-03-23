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

Each atomic task must:
- Start with a single action verb (add, create, write, configure, update, delete)
- Do exactly one thing — touches one file or one concern
- Have a short `title` (3-8 words) and a one-sentence `description` that says **what gets created and where** (target file, function name, or endpoint — enough for someone to implement it without re-reading the original prompt)
- Be assigned an `exec_order` (1, 2, 3...) reflecting dependency order

**Do not create separate test tasks.** The atomic skill (`/devkit:atomic`) already writes and runs tests as part of its own dev loop. Creating "write test for X" tasks leads to redundant work.

**Split any task that:**
- Contains "and" — it's two tasks
- Touches more than one file or layer
- Is vague ("set up X", "handle Y") — break it into concrete actions
- Would produce more than 100 lines of implementation code (tests excluded) — if you estimate it will exceed 100 lines, split it

If decomposition yields only one task, tell the user the task is already atomic. Offer two options: run `/devkit:atomic <description>` directly, or save it to the DB if they want it tracked.

If decomposition yields more than 20 tasks, the original prompt is likely too broad for a single decomposition. Suggest splitting it into 2-3 high-level milestones and running atomize on each one separately.

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

**Completeness check** — before saving, review the task list against the elaboration from Step 2. Every aspect of "done" must map to at least one task. If there's a gap, add the missing task(s). Show the final list to the user and get confirmation before proceeding.

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
" "the parent_prompt string from Step 4"
```

The ID column is what the user passes to `/devkit:atomic`. The atomic skill looks up the task by ID, reads its title and description, and uses them as the task definition for its dev loop.
