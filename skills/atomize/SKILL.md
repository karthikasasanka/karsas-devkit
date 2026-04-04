---
name: atomize
description: Decompose a task prompt into atomic subtasks and save to .devkit/tasks.db. Use whenever a user gives a large or vague task and wants it broken into smallest possible units of work before implementing.
argument-hint: "[--append] [--dry-run] [--tags tag1,tag2] [task description]"
---

## Step 0 — Validate input

Parse flags from `$ARGUMENTS` first:

- `--append` — add new tasks without clearing existing tasks for this `parent_prompt`
- `--dry-run` — print the task list without saving to DB; skip Step 4
- `--tags <value>` — comma-separated tags to attach to all tasks (e.g. `--tags backend,api`)

Strip parsed flags; the remainder is the task description.

If the description is empty after stripping, stop and ask the user to provide one.

If `$ARGUMENTS` references an external issue tracker (Jira key, GitHub issue `#123`, Linear issue), fetch the full issue details — title, description, acceptance criteria. Use the fetched text as the task definition and as `parent_prompt` in Step 4. Verify it's complete before proceeding.

---

## Step 1 — Clarify unknowns

**Read project context** — check `CLAUDE.md` for documented resources (template directories, catalogs, schemas, APIs, conventions). Scan for files relevant to the task (templates/, schema files, seed data). Record what you find — these become constraints in Step 3.

**Infer technical decisions** — scan project files (`pyproject.toml`, `package.json`, `go.mod`, `Cargo.toml`, source code) to understand the stack. Only ask about what can't be inferred.

For remaining unspecified decisions that affect decomposition, ask about all of them in one grouped question:

- **Language/framework** — e.g. Python/FastAPI, Node/Express, Go
- **Database** — e.g. SQLite, Postgres, MongoDB, in-memory
- **Auth mechanism** — e.g. JWT, sessions, OAuth (only if auth is relevant)
- **Testing** — should tests be included? Which framework?
- **API style** — REST, GraphQL, RPC (only if an API is being built)

Skip questions that don't apply. If nothing is unclear, proceed to Step 2.

---

## Step 2 — Elaborate the task

Restate the original prompt as a detailed description covering: intent (end goal), assumptions being made, and what "done" looks like.

Show this as a short paragraph and confirm with the user before proceeding. Incorporate corrections and re-confirm. If alignment isn't converging after a few rounds, ask the user to restate in their own words.

Do not proceed to decomposition until the user agrees.

---

## Step 3 — Decompose into atomic tasks

Split the task into atomic units. A function is one named, testable unit that does one thing. Related functions that must exist together to be testable (e.g., a route handler and its validation schema) count as one logical unit.

**No self-referential test tasks.** The `atomic` skill already writes and runs tests in its own dev loop — don't create "write tests for X" tasks where X is another task in the same list. Exception: if the prompt is explicitly about testing (e.g., "write tests for the auth module"), test tasks are the deliverable.

Each task must:
- Start with a single action verb (add, create, write, configure, update, delete)
- Do exactly one thing — touches one file or one concern
- Have a short `title` (3-8 words) and a one-sentence `description` stating **what gets created and where**
- Have an `exec_order` (1, 2, 3...) reflecting dependency order
- Have a `size`: **S** (simple), **M** (moderate), or **L** (complex or multiple tightly related pieces)
- Have a `depends_on` list of exec_order values of prerequisite tasks (empty if none); must form a **DAG — no cycles**

**Split any task that:**
- Contains "and" — it's two tasks
- Touches more than one file or layer
- Is vague ("set up X", "handle Y") — break into concrete actions
- Would produce more than 100 lines of implementation code

**Edge cases:**
- Single task after decomposition → tell the user the task is already atomic. Offer to run `/devkit:atomic <description>` directly, or save to DB for tracking.
- More than 20 tasks → the prompt is too broad. Suggest splitting into 2-3 milestones and running atomize on each.
- Frontend tasks → component-level granularity, not function-level. A form with three fields is one task.

**Completeness check** — before saving, map each requirement from Step 2 to a task. Present a coverage table and get confirmation:

| Requirement | Covered by | Size |
|---|---|---|
| user can download completed document | #4 add PDF download button | S |
| form values substitute into template | #2 create NDA document component | M |

---

## Step 4 — Save to SQLite

If `--dry-run` was set, skip this step. Print the task list (including size, depends_on) and tell the user: "Dry run complete — nothing was saved."

Otherwise, save to `.devkit/tasks.db` using this Python script via `uv run python`:

```python
import sqlite3, os

# Populate with decomposed tasks from Step 3.
# Each dict must have: title, description, exec_order, size, depends_on_orders (list of exec_order ints).
tasks = [
    # {"title": "create users table migration", "description": "adds one schema migration file", "exec_order": 1, "size": "S", "depends_on_orders": []},
]

# Set to the original user prompt text (after stripping flags).
parent_prompt = ""
# Set to the --tags value, or "" if not provided.
tags_arg = ""
# Set to True if --append flag was provided.
append_mode = False

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

# Add new columns to existing DBs if missing
for col, defn in [
    ("depends_on", "TEXT DEFAULT NULL"),
    ("size", "TEXT DEFAULT NULL"),
    ("tags", "TEXT DEFAULT NULL"),
]:
    try:
        conn.execute(f"ALTER TABLE tasks ADD COLUMN {col} {defn}")
        conn.commit()
    except Exception:
        pass  # column already exists

if not append_mode:
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
else:
    max_order = conn.execute(
        "SELECT COALESCE(MAX(exec_order), 0) FROM tasks WHERE parent_prompt = ?", (parent_prompt,)
    ).fetchone()[0]
    for t in tasks:
        t["exec_order"] += max_order

order_to_id = {}
for t in tasks:
    cur = conn.execute(
        "INSERT INTO tasks (parent_prompt, title, description, exec_order, size, tags) VALUES (?, ?, ?, ?, ?, ?)",
        (parent_prompt, t["title"], t["description"], t["exec_order"], t.get("size"), tags_arg or None)
    )
    order_to_id[t["exec_order"]] = cur.lastrowid
conn.commit()

for t in tasks:
    dep_orders = t.get("depends_on_orders", [])
    if dep_orders:
        dep_ids = ",".join(str(order_to_id[o]) for o in dep_orders if o in order_to_id)
        if dep_ids:
            conn.execute("UPDATE tasks SET depends_on = ? WHERE id = ?", (dep_ids, order_to_id[t["exec_order"]]))
conn.commit()
conn.close()
print(f"Saved {len(tasks)} tasks to .devkit/tasks.db")
```

Write this as `.devkit/save_tasks.py`, populate the variables, run it with `uv run python .devkit/save_tasks.py`, then delete it. If the script exits non-zero, stop and show the error. All automation files must stay inside `.devkit/`.

After the script completes, update `.devkit/tasks.md`:
- **Not exists** — create with a header section for this decomposition.
- **Exists** — append a new section below existing content.

Format (use actual IDs from the DB, not list indices — query after saving):
```
# <first 80 chars of parent_prompt>

| ID | Order | Size | Title | Status |
|----|-------|------|-------|--------|
| 1  | 1     | M    | create users table migration | pending |
```

---

## Step 5 — Display result

```
uv run python -c "
import sqlite3, os, sys
db_path = '.devkit/tasks.db'
if not os.path.exists(db_path):
    print('ERROR: .devkit/tasks.db not found. Run atomize first.')
    raise SystemExit(1)
parent_prompt = sys.argv[1]
conn = sqlite3.connect(db_path)
rows = conn.execute('SELECT id, exec_order, size, title, depends_on, status FROM tasks WHERE parent_prompt = ? ORDER BY exec_order', (parent_prompt,)).fetchall()
conn.close()
print(f'{\"ID\":<5} {\"Order\":<7} {\"Size\":<6} {\"Title\":<40} {\"Depends On\":<15} {\"Status\"}')
print('-' * 85)
for r in rows:
    print(f'{r[0]:<5} {r[1]:<7} {r[2] or \"\":<6} {r[3]:<40} {r[4] or \"\":<15} {r[5]}')
print()
print(f'Total: {len(rows)} tasks')
print()
print('Use: /devkit:atomic <ID> to execute a task')
" "<the actual parent_prompt value from Step 4>"
```

The ID column is what the user passes to `/devkit:atomic`. The atomic skill looks up the task by ID, reads its title and description, and uses them as the task definition for its dev loop.
