---
name: status
description: Display the current state of all tasks in .devkit/tasks.db, grouped by status with counts. Use this to check where a build left off, see pending work between sessions, or get a quick overview of task progress without triggering any execution.
argument-hint: "[optional: parent_prompt filter]"
---

Read `.devkit/tasks.db` and display all tasks grouped by status (done / in_progress / pending) with counts and a compact table.

If `$ARGUMENTS` is provided, filter tasks to only those whose `parent_prompt` contains the argument as a substring. This lets you scope the view to a specific decomposition.

Run this query:

```python
uv run python -c "
import sqlite3, os, sys
from collections import Counter

db = '.devkit/tasks.db'
if not os.path.exists(db):
    raise SystemExit('No .devkit/tasks.db found. Run the atomize skill first.')

conn = sqlite3.connect(db)
filter_arg = sys.argv[1] if len(sys.argv) > 1 else ''
if filter_arg:
    rows = conn.execute(
        'SELECT id, exec_order, title, status, size, parent_prompt FROM tasks WHERE parent_prompt LIKE ? ORDER BY status, exec_order',
        (f'%{filter_arg}%',)
    ).fetchall()
else:
    rows = conn.execute(
        'SELECT id, exec_order, title, status, size, parent_prompt FROM tasks ORDER BY status, exec_order'
    ).fetchall()
conn.close()

if not rows:
    print('No tasks found.')
    raise SystemExit(0)

counts = Counter(r[3] for r in rows)
print(f'Total: {len(rows)} tasks — done={counts[\"done\"]}  in_progress={counts[\"in_progress\"]}  pending={counts[\"pending\"]}')
print()

for status in ['in_progress', 'pending', 'done']:
    group = [r for r in rows if r[3] == status]
    if not group:
        continue
    label = {'in_progress': 'IN PROGRESS', 'pending': 'PENDING', 'done': 'DONE'}[status]
    print(f'--- {label} ({len(group)}) ---')
    for r in group:
        print(f'  {r[0]:<5} order={r[1]:<4} size={r[4] or \"?\":<2}  {r[2]}')
    print()
" "$ARGUMENTS"
```

This skill is read-only — it has no side effects. Use it between sessions to see where a build left off without triggering any execution.
