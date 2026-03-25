# karsas-devkit

Claude Code plugin for clean, incremental development.

## Install

```
/plugin marketplace add https://github.com/karthikasasanka/karsas-devkit
/plugin install devkit@karsas-devkit
```

## Skills

### `atomize` — decompose a task

Break a large or vague task into atomic subtasks and save them to `.devkit/tasks.db`.

```
/atomize build a REST API for user authentication
```

### `build` — execute all pending tasks

Run all pending tasks from `.devkit/tasks.db` in order. Runs `atomic` on each task, then either waits for you to merge each PR (sequential mode) or collects all commits on one branch and opens a single PR at the end (batch mode).

```
/build
/build build a REST API for user authentication
```

Run `atomize` first if tasks haven't been decomposed yet.

### `atomic` — implement one task

Write one function, test it, commit if green, fix if red. Accepts a prompt or a task ID from `atomize`.

```
/atomic add a login route that validates JWT tokens
/atomic 3
```

When given a task ID, the skill marks it `in_progress` on start and `done` on commit.

## License

MIT
