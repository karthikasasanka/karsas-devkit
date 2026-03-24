---
name: build
description: Orchestrate sequential execution of all pending atomic tasks from .devkit/tasks.db. Runs the atomic skill on each task in exec_order, pushes a PR after each one, then pauses and waits for the user to merge before moving on. Use this whenever the user wants to run all their atomized tasks, execute the task queue, work through pending tasks one by one, or says things like "run all tasks", "execute the task list", "go through all the atomic tasks", "start the queue", or "build everything".
argument-hint: "[optional: parent_prompt filter to run only tasks from a specific decomposition]"
---

## Overview

This skill drives sequential execution of atomic tasks already saved in `.devkit/tasks.db`:

1. **Load** — query pending tasks in exec_order, confirm trunk branch and task list with user
2. **Execute** — run the `atomic` skill on each task
3. **Gate** — after each PR is pushed, pause for the user to merge before continuing

Run `atomize` first if tasks haven't been decomposed yet. This skill only executes — it does not decompose.

---

## Step 1 — Setup

### 1a. Ask about trunk branch

Ask the user once: "What is your trunk branch — `main` or `master`?" Remember the answer and use it for all git syncs in this session.

### 1b. Load the pending task list

Query `.devkit/tasks.db`. If `$ARGUMENTS` is provided and looks like a task description (not a number), filter by `parent_prompt`:

```python
uv run python -c "
import sqlite3, sys, os
db = '.devkit/tasks.db'
if not os.path.exists(db):
    raise SystemExit('No .devkit/tasks.db found. Run the atomize skill first to create tasks.')
conn = sqlite3.connect(db)
filter_arg = sys.argv[1] if len(sys.argv) > 1 else ''
if filter_arg:
    rows = conn.execute(
        'SELECT id, exec_order, title, status FROM tasks WHERE parent_prompt LIKE ? ORDER BY exec_order',
        (f'%{filter_arg}%',)
    ).fetchall()
else:
    rows = conn.execute(
        'SELECT id, exec_order, title, status FROM tasks ORDER BY exec_order'
    ).fetchall()
conn.close()
pending = [r for r in rows if r[3] in ('pending', 'in_progress')]
print(f'{\"ID\":<5} {\"Order\":<7} {\"Title\":<45} {\"Status\"}')
print('-' * 70)
for r in rows:
    print(f'{r[0]:<5} {r[1]:<7} {r[2]:<45} {r[3]}')
print()
print(f'Pending: {len(pending)} of {len(rows)} tasks')
" "$ARGUMENTS"
```

From the results:
- **Skip** tasks with status `done` — already finished.
- **Treat `in_progress` as pending** — likely a prior interrupted session. The atomic skill will handle the resume/restart dialogue when it picks up that task; let it do so without intervening.
- **Stop** if there are no pending tasks — tell the user everything is already done.

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

The atomic skill handles everything: feature branch, agent pipeline, quality gates, commits, simplify, pr-review, and pushing the PR. Do not implement anything yourself — let atomic drive all of this.

**What to expect during atomic**: Near the end, atomic will ask the user for push confirmation before opening the PR — that is atomic's own confirmation step, not this skill's gate. Let the user respond to it normally.

**If atomic stops due to a failure** (e.g., quality gates still failing after 3 fix cycles, or a blocker the user needs to resolve): pause the build loop and ask the user whether to skip this task and continue to the next, retry the same task, or abort the run entirely. Do not proceed automatically.

### 2c. Wait for merge

After atomic reports the PR is open, pause:

```
Task <id> PR is open. Merge it on GitHub, then say "continue" or "next" to proceed.
Next up: <next-id>: <next-title>
```

(Omit the "Next up" line if this is the last task.)

Do not proceed until the user explicitly confirms the PR is merged.

### 2d. Sync trunk

Once the user confirms:

```bash
git fetch origin
git checkout <trunk-branch>
git pull
```

Then move to the next task.

---

## Step 3 — Final summary

After all tasks are done, re-query the DB to get current statuses:

```python
uv run python -c "
import sqlite3, sys, os
conn = sqlite3.connect('.devkit/tasks.db')
filter_arg = sys.argv[1] if len(sys.argv) > 1 else ''
if filter_arg:
    rows = conn.execute(
        'SELECT id, title, status FROM tasks WHERE parent_prompt LIKE ? ORDER BY exec_order',
        (f'%{filter_arg}%',)
    ).fetchall()
else:
    rows = conn.execute('SELECT id, title, status FROM tasks ORDER BY exec_order').fetchall()
conn.close()
for r in rows:
    print(f'{r[0]:<5} {r[1]:<45} {r[2]}')
" "$ARGUMENTS"
```

Print the table and a count of how many tasks were completed in this session (exclude tasks that were already `done` before this run started).

If `.devkit/tasks.md` exists, remind the user it contains per-task completion notes written during each atomic run.
