---
name: atomize
description: Decompose a task prompt into atomic subtasks and save to .devkit/tasks.db. Use whenever a user gives a large or vague task and wants it broken into smallest possible units of work before implementing.
argument-hint: "[task description]"
---

Task: $ARGUMENTS

---

## Step 1 — Elaborate the task

Restate the original prompt as a detailed description:
- Clarify intent: what is the end goal?
- List assumptions being made
- Identify any unknowns that would need resolving before implementation
- Define what "done" looks like

Output this as a short paragraph before proceeding.

---

## Step 2 — Decompose into atomic tasks

Split the task into atomic units. Each atomic task must:
- Start with a single action verb (add, create, write, configure, update, delete)
- Do exactly one thing — touches one file or one concern
- Match the `atomic` skill's definition: one named, testable unit
- Have a short `title` (5–8 words) and a one-sentence `description`
- Be assigned an `exec_order` (1, 2, 3...) reflecting dependency order

**Split any task that:**
- Contains "and" — it's two tasks
- Touches more than one file or layer
- Is vague ("set up X", "handle Y") — break it into concrete actions

**Good atomic tasks:**
- `create users table migration` — adds one schema migration file
- `write get-user-by-id query` — one function, one SQL query
- `add POST /users route handler` — one route, one handler function

**Bad (split these):**
- `set up database and add user model` → two tasks
- `implement auth` → too vague, split into: create JWT signing function, create token validation middleware, add login route handler

---

## Step 3 — Save to SQLite

Create `.devkit/` in the current working directory if it doesn't exist.

Save all tasks to `.devkit/tasks.db` using this Python script via `uv run python`:

```python
import sqlite3, os, sys
from datetime import datetime

tasks = []  # populated below

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
for t in tasks:
    conn.execute(
        "INSERT INTO tasks (parent_prompt, title, description, exec_order) VALUES (?, ?, ?, ?)",
        (t["parent_prompt"], t["title"], t["description"], t["exec_order"])
    )
conn.commit()
conn.close()
print(f"Saved {len(tasks)} tasks to .devkit/tasks.db")
```

Write this as a temporary script, populate the `tasks` list with the decomposed tasks from Step 2, run it with `uv run python <script>`, then delete the script.

---

## Step 4 — Display result

After saving, query and display all tasks:

```
uv run python -c "
import sqlite3
conn = sqlite3.connect('.devkit/tasks.db')
rows = conn.execute('SELECT id, exec_order, title, status FROM tasks ORDER BY exec_order').fetchall()
conn.close()
print(f'{'ID':<4} {'Order':<6} {'Title':<40} {'Status'}')
print('-' * 65)
for r in rows:
    print(f'{r[0]:<4} {r[1]:<6} {r[2]:<40} {r[3]}')
"
```
