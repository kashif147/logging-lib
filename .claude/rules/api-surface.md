# API surface

Everything is exported from the single `module.exports` at the bottom of `index.js`: `createLogger`,
`resolveLogRoot`, `correlationIdMiddleware`, `logErrorMiddleware`, `createSystemLogsRouter`,
`createRabbitStructuredLogHandlers`, `requestContext`, `extractPayloadBusinessIds`.

**Record shape** (`finalizeRecord`): every log line is single-line JSON with a fixed set of
top-level fields always present (`timestamp`, `level`, `service`, `message`, `correlationId`,
`userId`, `tenantId`, `profileId`, `applicationId`, `membershipId`, `eventType` — `null` when absent,
never omitted), plus any additional `meta` keys spread on top *without overwriting* the fixed fields
(see the `hasOwnProperty` guard in the `rest` loop). If you add a new well-known field that every
service should be able to rely on being present, add it to `finalizeRecord`'s destructuring, not just
to individual call sites — a field added only at a call site is `null` everywhere else unless that
caller happens to pass it.

**`req` is optional sugar, not a requirement**: `bizLogger.business(message, meta, req)` and
`.error(...)` accept an Express `req` as a third argument purely to auto-populate
`correlationId`/`userId`/`tenantId` from it (`requestContext()` reads `req.correlationId` /
`x-correlation-id`, `x-user-id` or `req.user.id`/`_id`, `req.tenantId` or `x-tenant-id`) — `meta`
passed explicitly always wins over what `req` would have inferred (`{...ctx, ...meta}` merge order).
Every caller in this codebase and consuming services omits `req` for background/RabbitMQ contexts and
passes it for Express request handlers — follow that split rather than passing `req` everywhere or
nowhere.

**RabbitMQ hook shape**: `createRabbitStructuredLogHandlers(bizLogger)` returns `{onPublish,
onConsume, onFail, onDlq}`, meant to be passed as the `structuredLog` option into
`@projectShell/rabbitmq-middleware`'s `init({ structuredLog: ... })` — see any service's
`rabbitMQ/index.js` for the wiring. Each hook maps to one of `bizLogger.rabbitPublished/
rabbitConsumed/rabbitFailed/rabbitDlq`, all four of which independently duplicate the same
`correlationId`/`eventType`/`profileId`/`applicationId`/`membershipId` defaulting logic that
`finalizeRecord` already does — if you refactor one, check the other three stayed consistent.
`extractPayloadBusinessIds(payload)` is a separate helper for pulling those same three IDs out of a
raw event payload's `data` (or top-level) shape when a consumer needs to log before/without calling
one of the `rabbit*` methods directly.

**`logErrorMiddleware(bizLogger)`** returns an Express 4-arg error-handling middleware — it logs and
then always calls `next(err)`, so it must be mounted *before* a service's own terminal error handler
(see the wiring order in the README's Express example). It is not a replacement for that terminal
handler; mounting it last will swallow the error response.
