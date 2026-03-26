---
name: atomize
description: Decompose a task prompt into atomic subtasks and save to .devkit/tasks.db. Use whenever a user gives a large or vague task and wants it broken into smallest possible units of work before implementing.
argument-hint: "[--append] [--dry-run] [--tags tag1,tag2] [task description]"
---

## Step 0 — Validate input

Parse flags from `$ARGUMENTS` before treating the rest as the task description:

- `--append` — add new tasks to an existing decomposition without clearing any existing tasks for this `parent_prompt`
- `--dry-run` — print the decomposed task list to stdout without saving anything to the DB; skip Step 4
- `--tags <value>` — a comma-separated list of tags to attach to all tasks in this decomposition (e.g. `--tags backend,api`)

Strip parsed flags from `$ARGUMENTS` so the remainder is the task description.

If the task description is empty after stripping flags, stop and ask the user to provide one.

If `$ARGUMENTS` references an external issue tracker (Jira key like `PL-2`, GitHub issue `#123`, Linear issue), fetch the full issue details — title, description, and any acceptance criteria. Use this fetched text as the task definition for all steps below. The full text (not the issue key) becomes `parent_prompt` in Step 4. Verify the fetched text is complete before proceeding.

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

Only ask about decisions that change how the task decomposes. Skip questions that don't apply. If nothing is unclear, proceed directly to Step 2.

---

## Step 2 — Elaborate the task

Restate the original prompt as a detailed description:
- Clarify intent: what is the end goal?
- List assumptions being made
- Define what "done" looks like

Output this as a short paragraph and show it to the user for confirmation before proceeding. If the user has corrections, incorporate them and re-confirm. If after a few rounds alignment still isn't converging, suggest the user restate the task in their own words.

Do not proceed to decomposition until the user agrees the elaboration is accurate.

---

## Step 3 — Decompose into atomic tasks

Split the task into atomic units. A function is one named, testable unit that does one thing. When related functions must exist together to be testable (e.g., a route handler and its validation schema), they count as one logical unit.

**No self-referential test tasks.** The `/devkit:atomic` skill already writes and runs tests as part of its own dev loop. Do not create tasks like "write tests for X" where X is another task in the same list — that's redundant work. Before saving, scan the list and remove any task that exists only to test an implementation task within the same decomposition.

Exception: if the user's prompt is explicitly about testing (e.g., "write tests for the auth module"), then test tasks are the deliverable.

Each implementation task must:
- Start with a single action verb (add, create, write, configure, update, delete)
- Do exactly one thing — touches one file or one concern
- Have a short `title` (3-8 words) and a one-sentence `description` that says **what gets created and where**
- Be assigned an `exec_order` (1, 2, 3...) reflecting dependency order
- Be assigned a `size`: **S** (simple, single function or small component), **M** (moderate complexity), or **L** (complex or multiple tightly related pieces)
- Have a `depends_on` list — the `exec_order` values of tasks that must complete before this one (empty list if none). Model this as a **DAG — no cycles allowed**. Most tasks will have an empty `depends_on`.

**Split any task that:**
- Contains "and" — it's two tasks
- Touches more than one file or layer
- Is vague ("set up X", "handle Y") — break it into concrete actions
- Would produce more than 100 lines of implementation code

If decomposition yields only one task, tell the user the task is already atomic. Offer two options: run `/devkit:atomic <description>` directly, or save it to the DB if they want it tracked.

If decomposition yields more than 20 tasks, the original prompt is likely too broad. Suggest splitting it into 2-3 high-level milestones and running atomize on each separately.

**Right granularity for frontend tasks** — frontend tasks should be component-level, not function-level. A form with three fields is one task, not three.

**Completeness check** — before saving, list each requirement from the Step 2 elaboration and show which task covers it. A requirement with no covering task is a gap; add a task for it before saving. Present this as a coverage table alongside the task list:

| Requirement | Covered by | Size |
|---|---|---|
| user can download completed document | #4 add PDF download button | S |
| form values substitute into template | #2 create NDA document component | M |

Get the user's confirmation before saving.

---

## Step 4 — Save to SQLite

If `--dry-run` was set in Step 0, skip this step entirely. Print the task list from Step 3 (including size, depends_on) and tell the user: "Dry run complete — nothing was saved."

Otherwise, save all tasks to `.devkit/tasks.db` using this Python script via `uv run python`:

```python
import sqlite3, os

# Populate this list with decomposed tasks from Step 3.
# Each task dict must have: title, description, exec_order, size, depends_on_orders (list of exec_order ints).
tasks = [
    # {"title": "create users table migration", "description": "adds one schema migration file", "exec_order": 1, "size": "S", "depends_on_orders": []},
]

# Set this to the original user prompt text (after stripping flags).
parent_prompt = ""
# Set this to the --tags value, or "" if not provided.
tags_arg = ""
# Set this to True if --append flag was provided.
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
    # In append mode, find the max exec_order for this parent_prompt and offset new tasks
    max_order = conn.execute(
        "SELECT COALESCE(MAX(exec_order), 0) FROM tasks WHERE parent_prompt = ?", (parent_prompt,)
    ).fetchone()[0]
    for t in tasks:
        t["exec_order"] += max_order

# Insert tasks and track exec_order → id mapping for depends_on translation
order_to_id = {}
for t in tasks:
    cur = conn.execute(
        "INSERT INTO tasks (parent_prompt, title, description, exec_order, size, tags) VALUES (?, ?, ?, ?, ?, ?)",
        (parent_prompt, t["title"], t["description"], t["exec_order"], t.get("size"), tags_arg or None)
    )
    order_to_id[t["exec_order"]] = cur.lastrowid
conn.commit()

# Translate depends_on from exec_order references to actual DB IDs
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

Write this as `.devkit/save_tasks.py`, populate the variables, run it with `uv run python .devkit/save_tasks.py`, then delete it. If the script exits with a non-zero status, stop and show the error. All automation files must stay inside `.devkit/`.

After the script completes successfully, update `.devkit/tasks.md`:

- **If tasks.md does not exist** — create it with a header section for this decomposition.
- **If tasks.md already exists** — append a new section below the existing content.

The header section format (use actual IDs from the DB, not the list index — query after saving):

```
# <first 80 chars of parent_prompt>

| ID | Order | Size | Title | Status |
|----|-------|------|-------|--------|
| 1  | 1     | M    | create users table migration | pending |
...
```

---

## Step 5 — Display result

After saving, run this exact query and print the table:

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
