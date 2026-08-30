# Architecture

## Current structure
The application is intentionally small and static:

- `index.html` contains all HTML, CSS and browser JavaScript.
- The page fetches circular data from the configured API endpoint and renders cards client-side.
- A modal shows metadata, AI summary, key points and `testo_completo` when present.
- Attachment downloads use the configured download webhook and `mail_id`.
- `Dockerfile` packages the static application for deployment.

There is currently no frontend framework, package manifest, transpiler or build pipeline.

## Change rule
Prefer local, minimal changes that preserve this architecture unless a task explicitly requires an architectural migration.
