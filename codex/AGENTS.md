## Debugy

This section is intended to live inside your repository AGENTS.md alongside the rest of your project guidance. Merge it instead of replacing unrelated instructions.

Use Debugy when static code reading is not enough and you need runtime data.

## How it works

`debugy.log()` is the only log source. Logs are sent to Debugy Cloud.

- Set `DEBUGY_WRITE_KEY` in the app env → logs POST to Debugy Cloud
- If `DEBUGY_WRITE_KEY` is not set, logging is disabled (no-op)

## Logger template

This module is **server-only**. Never import it from client/browser code. If you need logging from client code, route it through an API endpoint or server action. Never modify this module to make it client-compatible.

The example below is TypeScript. Adapt the language, file path, and idioms to match the project. If the framework provides a way to enforce server-only usage at build time, use it. If not, add a runtime check that throws when the module loads in a browser context.

```ts
// lib/debugy.ts — server-only
const DEBUGY_WRITE_KEY = process.env.DEBUGY_WRITE_KEY ?? "";
const DEBUGY_CLOUD_URL = "https://www.debugy.dev";

function log(file: string, fn: string, message: string, opts: { level?: string; duration_ms?: number; metadata?: Record<string, string | number | boolean | null> } = {}) {
  if (!DEBUGY_WRITE_KEY) return;

  const entry = {
    timestamp: new Date().toISOString(),
    level: opts.level ?? "info",
    file,
    fn,
    message,
    ...(opts.duration_ms != null && { duration_ms: opts.duration_ms }),
    ...(opts.metadata && { metadata: opts.metadata }),
  };

  fetch(`${DEBUGY_CLOUD_URL}/api/logs`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${DEBUGY_WRITE_KEY}`,
    },
    body: JSON.stringify(entry),
  }).catch(() => {});
}

export const debugy = { log };
```

If the project uses a different language, implement the same behavior — keeping the logger server-side. For services in other languages, a simple HTTP POST is enough:

```
POST https://www.debugy.dev/api/logs
Authorization: Bearer <DEBUGY_WRITE_KEY>
Content-Type: application/json

{"timestamp":"...","level":"info","file":"worker.py","fn":"process","message":"done"}
```

**Browser logs:** Route client-side log calls through a server action or API endpoint that calls `debugy.log()` on the server. Never expose `DEBUGY_WRITE_KEY` or `DEBUGY_AGENT_KEY` to the client.

## Setup

1. Create a free account at https://www.debugy.dev/sign-up to get your `DEBUGY_WRITE_KEY` and `DEBUGY_AGENT_KEY`.
2. Create the `lib/debugy.ts` helper using the template above.
3. Set `DEBUGY_WRITE_KEY=dbg_pk_...` in the app env.
4. Keep `DEBUGY_AGENT_KEY=dbg_...` with the agent or local shell — not in the deployed app runtime.

## After install

Once setup is complete, scan the repo and add `debugy.log()` calls wherever they would give valuable runtime visibility — API route handlers, database calls, authentication flows, error boundaries, external service integrations, etc. Use your judgment on what matters for this specific codebase. These are permanent observability logs, not temporary debug calls.

Never log sensitive data — no passwords, tokens, API keys, session IDs, emails, or any PII. When you need context about a value, log its shape, not the value itself (e.g. `userId length: 36`, not the actual userId).

Also set up a global unhandled-error hook so unexpected errors are automatically captured. Use the runtime's built-in mechanism — adapt to whatever stack the project uses. Two examples:

**Node.js / Next.js** (in `instrumentation.ts` or server entry):
```ts
process.on("unhandledRejection", (reason) => {
  debugy.log("global", "unhandledRejection", String(reason), { level: "error" });
});
```

**Python** (in entry point):
```python
import sys
_default = sys.__excepthook__
def _handle(exc_type, exc_value, exc_tb):
    debugy.log("global", "unhandledException", str(exc_value), level="error")
    _default(exc_type, exc_value, exc_tb)
sys.excepthook = _handle
```

For other stacks, find the equivalent global hook (`recover` in Go, `at_exit` in Ruby, `window.onunhandledrejection` routed through a server endpoint for browser code, etc.). If the framework already has its own error handler, hook into that instead of fighting it.

## Workflow

1. If more visibility is needed, add `debugy.log()` calls at targeted spots
2. Ask the user to reproduce the issue
3. The user tells you to read logs by saying `debugy`:
   - Fetch cloud logs using `DEBUGY_AGENT_KEY` (check `.env`, `.env.local`, or the shell environment for the key)
4. Diagnose, fix, and verify
5. Remove temporary `debugy.log()` calls when done

## Reading logs (`debugy`)

Look for `DEBUGY_AGENT_KEY` in `.env`, `.env.local`, or the shell environment. If not found, tell the user to create a free account at https://www.debugy.dev/sign-up to get their keys.

You can filter by `level`, `file`, `fn`, `search`, `limit`, and `offset`:

```bash
# all recent logs
curl -s "https://www.debugy.dev/api/logs?limit=100" -H "Authorization: Bearer <DEBUGY_AGENT_KEY>"
# errors only
curl -s "https://www.debugy.dev/api/logs?level=error&limit=100" -H "Authorization: Bearer <DEBUGY_AGENT_KEY>"
# specific file
curl -s "https://www.debugy.dev/api/logs?file=checkout.ts&limit=50" -H "Authorization: Bearer <DEBUGY_AGENT_KEY>"
# specific function
curl -s "https://www.debugy.dev/api/logs?fn=handlePayment&limit=50" -H "Authorization: Bearer <DEBUGY_AGENT_KEY>"
# clear logs
curl -X DELETE "https://www.debugy.dev/api/logs?project_id=<PROJECT_ID>" -H "Authorization: Bearer <DEBUGY_AGENT_KEY>"
```

## `debugy.log()` contract

```
debugy.log(file, fn, message, { level?, duration_ms?, metadata? })
```

- Fire-and-forget; never await in hot paths
- Never log secrets, tokens, credentials, or PII

## Rules

- `debugy.log()` is server-only — never import it from client/browser code, never modify the module to make it client-compatible
- Keep `DEBUGY_AGENT_KEY` with the agent, never in deployed app env
- Prefer high-signal logs with real values in `metadata`
- Remove temporary Debugy calls when debugging is complete
