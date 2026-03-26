---
name: reset
description: Reset one or more in_progress tasks back to pending and delete their stale feature branches so atomic can start fresh. Use when tasks are stuck in_progress after an interrupted session, a failed run, or a context reset.
argument-hint: "<task-id> [,task-id2,...]"
---

## Step 0 — Validate input

If `$ARGUMENTS` is empty, stop and ask the user for one or more task IDs (comma-separated).

Parse `$ARGUMENTS` as a comma-separated list of task IDs. Strip whitespace.

---

## Step 1 — Show current state

Before making any changes, query and display the current status of the specified tasks:

```python
uv run python -c "
import sqlite3, os, sys
db = '.devkit/tasks.db'
if not os.path.exists(db):
    raise SystemExit('No .devkit/tasks.db found.')
ids = [i.strip() for i in sys.argv[1].split(',')]
conn = sqlite3.connect(db)
placeholders = ','.join('?' * len(ids))
rows = conn.execute(
    f'SELECT id, title, status FROM tasks WHERE id IN ({placeholders})',
    ids
).fetchall()
conn.close()
for r in rows:
    print(f'  Task {r[0]}: {r[1]} — {r[2]}')
" "$ARGUMENTS"
```

If any of the specified task IDs are not found, tell the user which ones are missing.

---

## Step 2 — Reset tasks

For each task in the provided list:

1. **Skip `done` tasks** — resetting a completed task's DB status would not undo committed code, so it's explicitly out of scope. If the user provides a `done` task ID, tell them: "Task <id> is already done — skipping. Resetting a completed task's status would not undo any committed code."

2. **Reset `in_progress` tasks to `pending`**:
   ```
   uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ? AND status = ?', ('pending', <id>, 'in_progress')); conn.commit(); conn.close(); print('Task <id> reset to pending.')"
   ```

3. **Delete the associated feature branch** — look for a branch named `feat/<slugified-title>` (same slug strategy as the atomic skill: lowercase, hyphenated, filler words stripped, max 40 chars). If the branch exists, delete it:
   ```bash
   git branch -D feat/<branch-name> 2>/dev/null && echo "Deleted branch feat/<branch-name>" || echo "Branch feat/<branch-name> not found — nothing to delete"
   ```
   If the branch has a remote tracking ref, also delete it:
   ```bash
   git push origin --delete feat/<branch-name> 2>/dev/null || true
   ```

---

## Step 3 — Confirm

Print a summary of what was done:

```
Reset complete:
  Task 3: create payment handler — status: pending, branch deleted
  Task 5: add user auth middleware — status: pending, branch not found (already deleted or never created)
  Task 7: create index page — done task, skipped

Use /devkit:atomic <id> or /devkit:build to re-run.
```
