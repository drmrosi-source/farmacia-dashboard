# Constraints

- Preserve existing working behavior unless the task explicitly changes it.
- Keep changes minimal in the current static HTML/CSS/JavaScript architecture.
- Do not introduce a framework, package manager or build system for a small change.
- Preserve the configured API/download integration unless the task explicitly changes it.
- Treat API-provided text as untrusted content and keep escaping/safe text insertion intact.
- Do not add credentials or secrets to the repository.
- The worker does not decide whether a task passed; trusted verification runs after worker execution.
