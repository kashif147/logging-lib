# Log destination resolution (`resolveLogRoot`)

If `LOG_ROOT` is set it **must be an absolute path** — matching a Docker bind mount's right-hand side
(`../../logs:/var/log/projectshell` → `LOG_ROOT=/var/log/projectshell`), not a relative path, which
resolves under the container's `cwd` and silently never lands on the host bind mount. A relative
`LOG_ROOT` triggers a one-time `console.warn`, not an error — don't turn this into a hard failure
without checking whether any service currently relies on the lenient behavior. Unset `LOG_ROOT` falls
back to `process.cwd()/logs`.

Every service's actual logs land at `<LOG_ROOT>/<serviceName>/{app,error}-<date>.log`, where
`serviceName` comes from `createLogger(name)`'s argument or `process.env.SERVICE_NAME`, sanitized via
`normalizeServiceName` (non `\w.-` characters become `_`).

In the full projectShell checkout (not a standalone clone of this repo), this rule is additionally
hook-enforced: `.claude/hooks/enforce-hard-rules.mjs` (root-level `PreToolUse` hook) blocks any edit
to a `.env*` file that sets `LOG_ROOT=` to a non-absolute path. That hook lives in the parent
checkout, so it has no effect when this package is cloned standalone — the check above in
`resolveLogRoot` is the only enforcement that travels with this repo itself.
