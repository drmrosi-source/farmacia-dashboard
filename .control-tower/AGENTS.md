# Agents

Workers are disposable and task-scoped.

- Read only the minimum project context needed for the task.
- Work only in the assigned Control Tower worktree.
- Preserve existing behavior outside the stated objective.
- Do not modify trusted policy to make verification easier.
- Do not add secrets, credentials or unrelated dependencies.
- Use the existing static architecture unless the task explicitly requires otherwise.
- Leave a concise handoff describing changed files, assumptions and any remaining risk.
