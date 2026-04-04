---
name: build
description: Orchestrate sequential execution of all pending atomic tasks from .devkit/tasks.db. Runs the atomic skill on each task in exec_order, pushes a PR after each one, then pauses and waits for the user to merge before moving on. Use this whenever the user wants to run all their atomized tasks, execute the task queue, work through pending tasks one by one, or says things like "run all tasks", "execute the task list", "go through all the atomic tasks", "start the queue", or "build everything".
argument-hint: "[--from <id>] [--only <ids>] [--status] [--retry <id>] [optional: parent_prompt filter]"
---

This skill drives sequential execution of atomic tasks from `.devkit/tasks.db`. It does not decompose tasks — run `atomize` first if needed.

---

## Step 0 — Parse flags

Parse from `$ARGUMENTS` (remainder is optional parent_prompt filter):

- `--status` — print status table and exit
- `--from <id>` — skip tasks with exec_order below the given task's exec_order
- `--only <ids>` — run only these task IDs (comma-separated)
- `--retry <id>` — reset task to `pending` and run only it

`--status` and `--retry` exit immediately after handling. `--from` and `--only` can combine with a parent_prompt filter.

### --status

```python
uv run python -c "
import sqlite3, os, sys
db = '.devkit/tasks.db'
if not os.path.exists(db):
    raise SystemExit('No .devkit/tasks.db found.')
conn = sqlite3.connect(db)
filter_arg = sys.argv[1] if len(sys.argv) > 1 else ''
if filter_arg:
    rows = conn.execute(
        'SELECT id, exec_order, title, status, size FROM tasks WHERE parent_prompt LIKE ? ORDER BY status, exec_order',
        (f'%{filter_arg}%',)
    ).fetchall()
else:
    rows = conn.execute(
        'SELECT id, exec_order, title, status, size FROM tasks ORDER BY status, exec_order'
    ).fetchall()
conn.close()
from collections import Counter
counts = Counter(r[3] for r in rows)
print(f'Status summary: done={counts[\"done\"]} in_progress={counts[\"in_progress\"]} pending={counts[\"pending\"]}')
print()
for status in ['in_progress', 'pending', 'done']:
    group = [r for r in rows if r[3] == status]
    if not group:
        continue
    print(f'--- {status.upper()} ({len(group)}) ---')
    for r in group:
        print(f'  {r[0]:<5} order={r[1]:<4} size={r[4] or \"?\":<2}  {r[2]}')
    print()
" "$ARGUMENTS"
```

### --retry

1. Reset: `uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('pending', <id>)); conn.commit(); conn.close()"`
2. Run: `/devkit:atomic <id>`
3. Exit.

---

## Step 1 — Setup

### 1a. Detect continue mode

```bash
git branch --show-current
git rev-parse --abbrev-ref origin/HEAD 2>/dev/null || echo "unknown"
```

- If `.devkit/batch.json` exists → read trunk and branch from it; skip 1b and 1c.
- If on a `feat/*` branch with `done` tasks → continuing. Infer trunk from `origin/HEAD` (strip `origin/`), default `main`. Tell user: "Continuing in batch mode on `<branch>` (trunk: `<trunk>`). Say 'switch to sequential' to change." Skip 1b and 1c.
- Otherwise → fresh start, ask 1b and 1c.

### 1b. Trunk branch (fresh start only)

Ask: "What is your trunk branch — `main` or `master`?"

### 1c. Delivery mode (fresh start only)

Ask: "How should tasks be delivered?"
- **Sequential** (default): one PR per task; wait for merge before next
- **Batch**: all tasks on one branch; one PR at the end

If **Batch**:
1. Derive branch name from first task title or parent_prompt (`feat/<name>`, max 40 chars). Confirm with user.
2. `git checkout -b feat/<name>`
3. Write `.devkit/batch.json`: `{"branch": "feat/<name>", "trunk": "<trunk>"}`

### 1d. Load and validate the pending task list

**HARD GATE — check before running anything else:**

```bash
test -f .devkit/tasks.db && echo "EXISTS" || echo "MISSING"
```

If `MISSING`: do NOT implement any tasks. Invoke `/devkit:atomize` with the full content of `$ARGUMENTS`, then re-run build from Step 1d.

```python
uv run python -c "
import sqlite3, sys, os
db = '.devkit/tasks.db'
if not os.path.exists(db):
    raise SystemExit('No .devkit/tasks.db found. Run the atomize skill first.')
conn = sqlite3.connect(db)
filter_arg = sys.argv[1] if len(sys.argv) > 1 else ''
if filter_arg:
    rows = conn.execute(
        'SELECT id, exec_order, title, status, size, depends_on FROM tasks WHERE parent_prompt LIKE ? ORDER BY exec_order',
        (f'%{filter_arg}%',)
    ).fetchall()
else:
    rows = conn.execute(
        'SELECT id, exec_order, title, status, size, depends_on FROM tasks ORDER BY exec_order'
    ).fetchall()
conn.close()
pending = [r for r in rows if r[3] in ('pending', 'in_progress')]
print(f'{\"ID\":<5} {\"Order\":<7} {\"Size\":<6} {\"Title\":<40} {\"Status\"}')
print('-' * 75)
for r in rows:
    print(f'{r[0]:<5} {r[1]:<7} {r[4] or \"\":<6} {r[2]:<40} {r[3]}')
print()
print(f'Pending: {len(pending)} of {len(rows)} tasks')
" "$ARGUMENTS"
```

Apply filters: `--from <id>` excludes tasks with lower exec_order; `--only <ids>` excludes unlisted IDs.

- Skip `done` tasks.
- Treat `in_progress` as pending (atomic handles resume).
- Stop if no pending tasks.

### 1e. Validate dependency graph

```python
uv run python -c "
import sqlite3
conn = sqlite3.connect('.devkit/tasks.db')
count = conn.execute('SELECT COUNT(*) FROM tasks WHERE depends_on IS NOT NULL AND status != \"done\"').fetchone()[0]
conn.close()
print(count)
"
```

If `0`, skip cycle check. Otherwise:

```python
uv run python -c "
import sqlite3
from collections import defaultdict, deque
conn = sqlite3.connect('.devkit/tasks.db')
rows = conn.execute('SELECT id, title, depends_on FROM tasks WHERE status != \"done\"').fetchall()
conn.close()
graph = defaultdict(list)
in_degree = {r[0]: 0 for r in rows}
id_to_title = {r[0]: r[1] for r in rows}
task_ids = set(r[0] for r in rows)
for task_id, title, depends_on in rows:
    if depends_on:
        for dep in depends_on.split(','):
            dep_id = int(dep.strip())
            if dep_id in task_ids:
                graph[dep_id].append(task_id)
                in_degree[task_id] = in_degree.get(task_id, 0) + 1
queue = deque(tid for tid in in_degree if in_degree[tid] == 0)
visited = 0
while queue:
    node = queue.popleft()
    visited += 1
    for neighbor in graph[node]:
        in_degree[neighbor] -= 1
        if in_degree[neighbor] == 0:
            queue.append(neighbor)
if visited != len(rows):
    cycle_tasks = [f'{tid}: {id_to_title[tid]}' for tid in in_degree if in_degree[tid] > 0]
    print('ERROR: Dependency cycle detected:')
    for t in cycle_tasks:
        print(f'  {t}')
    raise SystemExit(1)
else:
    print('Dependency graph OK.')
"
```

Abort if a cycle is detected. Do not proceed until the user fixes it.

### 1f. Confirm

If any tasks have `size`, print: `Tasks: 2S 3M 1L`

Show task list and confirm before proceeding.

---

## Step 2 — Execute loop

For each pending task in exec_order:

### 2a. Announce

```
Starting task <id> of <total pending>: <title>
```

### 2b. Run atomic

```
/devkit:atomic <task-id>
```

If atomic fails: ask to skip, retry, or abort. Track failures for final summary.

### 2c. After atomic completes

**Sequential**: pause after PR is open:
```
Task <id> PR is open. Merge it on GitHub, then say "continue" or "next".
Next up: <next-id>: <next-title>
```
Wait for user confirmation, then sync:
```bash
git fetch origin && git checkout <trunk> && git pull
```

**Batch**: no wait. Move to next task immediately.

---

## Step 2b — Batch finalize (batch mode only)

1. Run **pr-review-toolkit:review-pr** across all branch changes.
2. Confirm with user before pushing.
3. `git push -u origin <branch-name>`
4. Open one PR against trunk listing each task title as a bullet.
5. `rm .devkit/batch.json`

---

## Step 3 — Final summary

Re-query DB and print the status table.

If any tasks failed or were skipped this session:
```
Failed / skipped tasks:
  Task 4: create payment handler — gate failed. Use: /devkit:build --retry 4
  Task 6: add email notification — skipped by user. Use: /devkit:build --retry 6
```

If `.devkit/tasks.md` exists, remind the user it has per-task completion notes.
