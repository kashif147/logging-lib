# `createSystemLogsRouter`

Mounts `POST /api/system-logs` (consuming services mount it as
`app.use("/api", createSystemLogsRouter(bizLogger))`) as the n8n-workflow-callback ingest endpoint.

Validation is deliberately loose: the body just needs to be a JSON object with at least one
non-empty string among `eventType`, `workflowName`, `executionId`, `correlationId` — anything else
`400`s with `{ success: false, error, details }`.

Optional shared-secret gate via `SYSTEM_LOG_API_KEY` (compared against `x-system-log-key` or
`x-api-key`); when unset, the endpoint is open — that's the consuming service's configuration choice
to make, not something to change in this package.

Success is `204` with an empty body, not `200`/`201` — n8n workflows depend on that exact status and
empty body; don't "fix" it to a more conventional `200` without checking those workflows first.
