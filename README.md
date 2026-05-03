# @projectShell/logging-lib

Structured JSON logs (daily rotate), correlation IDs, RabbitMQ hooks, and an n8n ingest endpoint for Express microservices.

## Install

From each service `package.json` (HTTPS clone — works in Docker/CI without SSH):

```json
"@projectShell/logging-lib": "git+https://github.com/kashif147/logging-lib.git#main"
```

Pin a branch or tag after `#` (e.g. `#gateway`) for repeatable deploys. Then `npm install`. Docker/Alpine images need `git` available during `npm install` when using Git dependencies.

Optional — monorepo local checkout:

```json
"@projectShell/logging-lib": "file:../logging-lib"
```

Set `SERVICE_NAME` in the environment (for example in Docker) so log files land under `<LOG_ROOT>/<serviceName>/`.

### Where to set `LOG_ROOT` (compose)

Use the shared **`config/.env.common`** file (on a VM often **`/home/deploy/config/.env.common`** — only a **`config`** folder is required, not a `ProjectShell` directory).

1. Copy **`config/.env.common.example`** → **`config/.env.common`** and fill values.
2. Backend **`docker-compose.yml`** files load **`${ENV_COMMON_PATH:-../../config/.env.common}`** first, then `.env.staging`. That relative path works when **`config/`** sits next to each service directory (same on a flat VM deploy under **`/home/deploy/`**).
3. Put **`LOG_ROOT=/var/log/projectshell`** in **`config/.env.common`** so it matches the default bind mount **`../../logs:/var/log/projectshell`** (host dir **`logs/`** next to **`config/`** when services are **`/home/deploy/<service>/`**). Compose **`env_file`** injects this into the container; you do **not** need a duplicate **`environment: LOG_ROOT`** in **`docker-compose.yml`**.

If compose files are not two levels below **`config/`**, set an absolute path:

```bash
ENV_COMMON_PATH=/home/deploy/config/.env.common docker compose up -d
```

Keep **`LOG_ROOT`** aligned with the volume mount inside the container.

### Centralized logs (`LOG_ROOT`)

If **`LOG_ROOT`** is set (absolute path recommended in Docker, e.g. `/var/log/projectshell`), every service writes to **`${LOG_ROOT}/<serviceName>/`** instead of `<cwd>/logs/<serviceName>/`.

Mount one host directory on all containers and set the same `LOG_ROOT` so all microservices and gateway-security logs share one tree:

- **`${LOG_ROOT}/profile-service/app-YYYY-MM-DD.log`** (and `error-*.log`)
- **`${LOG_ROOT}/gateway/app-YYYY-MM-DD.log`** — `@membership/policy-middleware` gateway validation (`ERROR`/`WARN` lines map to app vs error logs via logging-lib)
- Optional: **`GATEWAY_LOG_SERVICE_NAME`** overrides the gateway folder name (default `gateway`).

If **`LOG_ROOT`** is unset, behavior stays **`process.cwd()/logs/<serviceName>/`** (per-process working directory).

Export **`resolveLogRoot()`** from this package if you need the resolved path in app code.

**Docker:** compose binds **`../../logs`** → **`/var/log/projectshell`** (on a VM typically **`/home/deploy/logs`** when services live under **`/home/deploy/<service>/`**); set **`LOG_ROOT=/var/log/projectshell`** in **`config/.env.common`**.

(`@membership/policy-middleware` gateway validation logs use the same `LOG_ROOT` under `gateway/` when services install that package.)

## Log files

- `<LOG_ROOT>/<serviceName>/app-YYYY-MM-DD.log` — business events (`logger.business`, RabbitMQ publish/consume success paths via hooks). Default `<LOG_ROOT>` is `logs` under `cwd` when env unset.
- `<LOG_ROOT>/<serviceName>/error-YYYY-MM-DD.log` — errors (`logger.error`, RabbitMQ failures/DLQ via hooks).

Entries are single-line JSON with `timestamp`, `level`, `service`, `message`, `correlationId`, optional `userId`, `tenantId`, `profileId`, `applicationId`, `membershipId`, `eventType`, plus any extra metadata.

Console output mirrors the same JSON lines (Docker/host logs stay available).

## Express wiring

```javascript
const bizLogger = require("./config/bizLogger.js"); // or createLogger(...) inline
const {
  correlationIdMiddleware,
  logErrorMiddleware,
  createSystemLogsRouter,
  createRabbitStructuredLogHandlers,
} = require("@projectShell/logging-lib");

app.use(correlationIdMiddleware); // early
app.use(express.json());
app.use("/api", createSystemLogsRouter(bizLogger)); // POST /api/system-logs
// ... routes ...
app.use(logErrorMiddleware(bizLogger)); // before your existing error handler
app.use(yourErrorHandler);
```

## RabbitMQ (`@projectShell/rabbitmq-middleware`)

Pass structured hooks when calling `init`:

```javascript
await init({
  url: process.env.RABBIT_URL,
  logger: console, // or your pino logger — unchanged
  structuredLog: createRabbitStructuredLogHandlers(bizLogger),
  serviceName: "my-service",
});
```

Payloads are enriched with `occurredAt`, `sourceService`, and UUID `eventId` while keeping existing fields such as `timestamp` and `data`.

## API helpers

```javascript
bizLogger.business("Application approved", {
  eventType: "ApplicationApproved",
  applicationId: "...",
  profileId: "...",
}, req); // optional req merges x-user-id, x-tenant-id, correlationId

bizLogger.error("Payment webhook failure", {
  eventType: "PaymentFailed",
  stack: err.stack,
}, req);

bizLogger.rabbitPublished({
  eventId, correlationId, exchange, queue, routingKey, eventType,
  retryCount: 0, profileId, applicationId, membershipId,
});
```

## n8n → `POST /api/system-logs`

Optional shared secret: set `SYSTEM_LOG_API_KEY` and send header `x-system-log-key` (or `x-api-key`). The body must be a JSON object with **at least one** non-empty string among: `eventType`, `workflowName`, `executionId`, or `correlationId`. Invalid bodies return **400** with `{ success: false, error, details }`; failures while writing logs return **500** with a generic message (no stack). Success returns **204** with an empty body.

Additional fields (`profileId`, `membershipId`, `status`, etc.) are merged into the business log line.

**Note:** `winston-daily-rotate-file` may flush log files slightly asynchronously right after startup; allow a short delay before tailing new files in smoke tests.

## HTTP noise

Per-request `console` logging is off unless `LOG_HTTP_REQUESTS=true` where that middleware was gated (for example profile-service).
