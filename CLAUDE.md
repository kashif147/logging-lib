# CLAUDE.md

`@projectShell/logging-lib` is the shared structured-logging package consumed by every backend
microservice in this platform (audit-service, communication-service, events-service, etc.) — see
their `package.json`s installing it as
`"@projectShell/logging-lib": "git+https://github.com/kashif147/logging-lib.git#main"` (or a pinned
branch/tag after `#`). It is intentionally a **single file**: `index.js`. There is no `src/`, no
build step, and no framework beyond `winston` + `winston-daily-rotate-file` + `uuid`. Every consuming
service picks up a change here on its next `npm install` against this git ref — a breaking change to
an exported function's signature is a breaking change across the whole platform simultaneously, not
just this repo.

`README.md` covers install/wiring examples in detail — read it first for how to use this package.
This file and its imports cover what the README doesn't: internal behavior other services are
implicitly relying on, most importantly the `LOG_ROOT` absolute-path requirement below.

## Commands

There is no `npm test`/`build`/`lint` script (`package.json` has no `scripts` block at all). The only
verification available is the manual smoke script:

```bash
node scripts/http-smoke.cjs
```

It boots a throwaway Express app on an ephemeral port, exercises `correlationIdMiddleware` and
`createSystemLogsRouter` end-to-end over real HTTP (not supertest), and either prints
`HTTP_CORRELATION_AND_SYSTEM_LOGS_OK` and exits 0, or `throw`s on the first mismatch (non-zero exit).
It writes to and cleans up `.tmp-http-smoke/` next to `scripts/`. Run this if you touch
`correlationIdMiddleware` or `createSystemLogsRouter` — it's the closest thing to a test suite this
package has.

### Log destination resolution (`LOG_ROOT`)
@.claude/rules/log-destination.md

### API surface
@.claude/rules/api-surface.md

### System logs router (n8n ingest endpoint)
@.claude/rules/system-logs-router.md
