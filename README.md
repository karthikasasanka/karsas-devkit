# karsas-devkit

Claude Code plugin for clean, incremental development.

## Install

```
/plugin marketplace add github:karthikasasanka/karsas-devkit
/plugin install devkit@karsas-devkit
```

## Skills

### `atomize` — decompose a task

Break a large or vague task into atomic subtasks and save them to `.devkit/tasks.db`.

```
/atomize build a REST API for user authentication
```

### `atomic` — implement one task

Write one function, test it, commit if green, fix if red. Accepts a prompt or a task ID from `atomize`.

```
/atomic add a login route that validates JWT tokens
/atomic 3
```

When given a task ID, the skill marks it `in_progress` on start and `done` on commit.

## License

MIT
