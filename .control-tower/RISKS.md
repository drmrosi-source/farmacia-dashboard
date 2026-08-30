# Risks

- The dashboard depends on external webhook availability and response shape.
- API fields may be missing or malformed; UI changes should retain defensive fallbacks.
- User-visible data is rendered dynamically; avoid introducing unsafe raw HTML insertion for API-provided text.
- The project has no browser test suite, so trusted verification currently checks static structure and JavaScript syntax rather than full UI behavior.
