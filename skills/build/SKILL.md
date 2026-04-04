---
name: build
description: Orchestrate sequential execution of all pending atomic tasks from .devkit/tasks.db. Runs the atomic skill on each task in exec_order, pushes a PR after each one, then pauses and waits for the user to merge before moving on. Use this whenever the user wants to run all their atomized tasks, execute the task queue, work through pending tasks one by one, or says things like "run all tasks", "execute the task list", "go through all the atomic tasks", "start the queue", or "build everything".
argument-hint: "[--from <id>] [--only <ids>] [--status] [--retry <id>] [optional: parent_prompt filter]"
---

## Overview

This skill drives sequential execution of atomic tasks already saved in `.devkit/tasks.db`:

1. **Load** — query pending tasks in exec_order, validate the dependency graph, confirm with user
2. **Execute** — run the `atomic` skill on each task
3. **Gate** — after each PR is pushed, pause for the user to merge before continuing

Run `atomize` first if tasks haven't been decomposed yet. This skill only executes — it does not decompose.

---

## Step 0 — Parse flags

Parse flags from `$ARGUMENTS` before treating the rest as an optional parent_prompt filter:

- `--status` — print the current task status table and exit immediately (no execution)
- `--from <id>` — skip all tasks with `exec_order` less than the exec_order of the given task ID; start from that task
- `--only <ids>` — run only the specified task IDs (comma-separated, e.g. `--only 3,5,7`); ignore all others
- `--retry <id>` — reset the specified task to `pending` and run only that task

These flags are mutually exclusive: if `--status` is present, just show status and exit. If `--retry` is present, handle the retry and exit. Otherwise, `--from` and `--only` can be combined with a parent_prompt filter.

### --status handling

If `--status` is set, query the DB and print a grouped status table, then exit:

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

### --retry handling

If `--retry <id>` is set:
1. Reset the task to `pending`:
   ```
   uv run python -c "import sqlite3; conn = sqlite3.connect('.devkit/tasks.db'); conn.execute('UPDATE tasks SET status = ? WHERE id = ?', ('pending', <id>)); conn.commit(); conn.close(); print('Task <id> reset to pending.')"
   ```
2. Run the atomic skill on that task: `/devkit:atomic <id>`
3. Exit after atomic completes.

---

## Step 1 — Setup

### 1a. Detect continue mode

Before asking anything, check whether this is a continuation of an in-progress build:

```bash
git branch --show-current
git rev-parse --abbrev-ref origin/HEAD 2>/dev/null || echo "unknown"
```

If the current branch is already a `feat/*` branch **and** there are `done` tasks in the DB, this is a continue session. In that case:
- Infer trunk from `git rev-parse --abbrev-ref origin/HEAD` (strip the `origin/` prefix). If ambiguous, default to `main`.
- If `.devkit/batch.json` exists, read trunk and branch from it — skip steps 1b and 1c entirely.
- If no `batch.json` but already on a `feat/*` branch with prior commits ahead of trunk, assume batch mode on the current branch. Tell the user: "Continuing in batch mode on `<branch>` (trunk: `<trunk>`). Say 'switch to sequential' to change." Then proceed without asking.

Only ask the questions in 1b/1c when starting fresh (no done tasks and not on a feature branch).

### 1b. Ask about trunk branch (fresh start only)

Ask the user once: "What is your trunk branch — `main` or `master`?" Remember the answer for all git syncs in this session.

### 1c. Choose delivery mode (fresh start only)

Ask the user: "How should tasks be delivered?"
- **Sequential** (default): one PR per task — each atomic task gets its own branch and PR; you merge each before the next task starts
- **Batch**: all tasks on one shared branch — each task commits to the same branch; one PR is opened at the end

If **Batch** is chosen:
1. Derive a branch name from the first task's title or the parent_prompt (lowercase, hyphenated, `feat/` prefix, max 40 chars). Show it to the user and allow them to rename it.
2. Create the branch: `git checkout -b feat/<name>`
3. Write `.devkit/batch.json`:
   ```json
   {"branch": "feat/<name>", "trunk": "<trunk-branch>"}
   ```

### 1d. Load and validate the pending task list

Query `.devkit/tasks.db`. Apply any `--from` or `--only` filters after loading:

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

Apply `--from <id>` filter: find the `exec_order` of that task ID and exclude tasks with lower exec_order.

Apply `--only <ids>` filter: exclude all tasks whose IDs are not in the provided list.

From the results:
- **Skip** tasks with status `done` — already finished.
- **Treat `in_progress` as pending** — the atomic skill will handle the resume/restart dialogue.
- **Stop** if there are no pending tasks — tell the user everything is already done.

### 1e. Validate dependency graph

Before confirming, check whether any pending task has a `depends_on` value:

```python
uv run python -c "
import sqlite3, os
db = '.devkit/tasks.db'
conn = sqlite3.connect(db)
count = conn.execute('SELECT COUNT(*) FROM tasks WHERE depends_on IS NOT NULL AND status != \"done\"').fetchone()[0]
conn.close()
print(count)
"
```

If the result is `0`, skip the cycle check entirely — there are no dependencies to validate.

Otherwise, validate that `depends_on` references form a DAG (no cycles) for the tasks about to run. Run this check:

```python
uv run python -c "
import sqlite3, os, sys
from collections import defaultdict, deque

db = '.devkit/tasks.db'
conn = sqlite3.connect(db)
rows = conn.execute('SELECT id, title, depends_on FROM tasks WHERE status != \"done\"').fetchall()
conn.close()

# Build graph
graph = defaultdict(list)
in_degree = {r[0]: 0 for r in rows}
id_to_title = {r[0]: r[1] for r in rows}
task_ids = set(r[0] for r in rows)

for task_id, title, depends_on in rows:
    if depends_on:
        for dep in depends_on.split(','):
            dep = dep.strip()
            if dep:
                dep_id = int(dep)
                if dep_id in task_ids:
                    graph[dep_id].append(task_id)
                    in_degree[task_id] = in_degree.get(task_id, 0) + 1

# Kahn's algorithm
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
    print('ERROR: Dependency cycle detected in the following tasks:')
    for t in cycle_tasks:
        print(f'  {t}')
    raise SystemExit(1)
else:
    print('Dependency graph OK — no cycles detected.')
"
```

If a cycle is detected, abort immediately. Do not proceed until the user fixes the dependencies (either by re-running atomize or manually editing the DB).

### 1e. Show size summary and confirm

If any tasks have a `size` value, print a one-line summary before asking for confirmation:

```
Tasks: 2S 3M 1L  (or however many of each)
```

Show the user the task list and confirm before proceeding.

---

## Step 2 — Execute loop

For each pending task in exec_order:

### 2a. Announce

```
Starting task <id> of <total pending>: <title>
```

### 2b. Run atomic

Invoke the **atomic** skill with the task's numeric ID:

```
/devkit:atomic <task-id>
```

The atomic skill handles everything: feature branch, agent pipeline, quality gates, commits, simplify, pr-review, and pushing the PR.

**If atomic stops due to a failure**: pause the build loop and ask the user whether to skip this task and continue, retry the same task, or abort the run entirely.

**Track failures**: keep a list of tasks that failed or were skipped (with reason) for the final summary.

### 2c. After atomic completes

**Sequential mode**: After atomic reports the PR is open, pause:
```
Task <id> PR is open. Merge it on GitHub, then say "continue" or "next" to proceed.
Next up: <next-id>: <next-title>
```
(Omit "Next up" if this is the last task.) Do not proceed until the user explicitly confirms the PR is merged. Then sync trunk:
```bash
git fetch origin
git checkout <trunk-branch>
git pull
```

**Batch mode**: No wait, no sync. Move directly to the next task.

---

## Step 2b — Batch finalize (batch mode only)

After all tasks complete, run these in order:

1. **PR review** — run **pr-review-toolkit:review-pr** across all changes on the branch.
2. **Confirm with user** — ask before pushing.
3. **Push**: `git push -u origin <branch-name>`
4. **Open one PR** against trunk summarizing all tasks implemented. List each task title as a bullet.
5. **Delete batch state**: `rm .devkit/batch.json`

---

## Step 3 — Final summary

After all tasks are done, re-query the DB to get current statuses and print the table.

If any tasks were skipped or failed during this run, print a failure summary:

```
Failed / skipped tasks:
  Task 4: create payment handler — gate failed after 3 fix cycles. Use: /devkit:build --retry 4
  Task 6: add email notification — skipped by user. Use: /devkit:build --retry 6
```

Print only tasks that failed or were skipped in this session (not previously failed ones). If all tasks completed successfully, omit this section.

If `.devkit/tasks.md` exists, remind the user it contains per-task completion notes written during each atomic run.
