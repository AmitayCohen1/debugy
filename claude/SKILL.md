---
name: debugy
description: Runtime debugging workflow for Debugy.
allowed-tools: Bash(node *) Read Write Edit Glob Grep
---

# Debugy - Local Runtime Visibility

Treat Debugy as a reusable Claude skill. Use it when static code reading is not enough and you need runtime evidence from the app.

## How It Works

Logs are written to `.debugy/session.ndjson` — one JSON object per line. The agent reads this file to get runtime evidence. No server, no API keys, no network.

## Setup

Add the logger to the project. Here's a TypeScript example — if the project uses a different language, implement the same behavior in that language:

```ts
// lib/debugy.ts
const DEBUGY_ENV = process.env.DEBUGY_ENV ?? "";

function log(file: string, fn: string, message: string, opts: { level?: string; duration_ms?: number; metadata?: Record<string, unknown> } = {}) {
  const entry = {
    timestamp: new Date().toISOString(),
    level: opts.level ?? "info",
    file,
    fn,
    message,
    env: DEBUGY_ENV,
    ...(opts.duration_ms != null && { duration_ms: opts.duration_ms }),
    ...(opts.metadata && { metadata: opts.metadata }),
  };

  // No DEBUGY_ENV set → do nothing (safe in production)
  if (!DEBUGY_ENV) return;
  // In development → write to local file. Other envs need cloud setup.
  if (DEBUGY_ENV !== "development") return;

  if (typeof window !== "undefined") {
    // Browser: POST to local dev server
    fetch("/api/debugy", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(entry),
    }).catch(() => {});
  } else {
    // Node.js: write directly to file
    try {
      const fs = require("fs");
      fs.mkdirSync(".debugy", { recursive: true });
      fs.appendFileSync(".debugy/session.ndjson", JSON.stringify(entry) + "\n");
    } catch {}
  }
}

export const debugy = { log };
```

**Client-side logs (optional):** If the project runs code in the browser, create a local API route so client logs reach the same file. The route just appends to `.debugy/session.ndjson`. Use whatever convention your framework supports — for example:

- **Next.js**: `app/api/debugy/route.ts`
- **Express**: `app.post("/api/debugy", ...)`
- **SvelteKit**: `src/routes/api/debugy/+server.ts`

The handler is the same for all — read the JSON body, append it to `.debugy/session.ndjson`.

If the project has no backend (static site, SPA without API), skip this — the agent can add server-side `debugy.log()` calls or read browser console output from the terminal instead.

Make sure `.debugy/` is in `.gitignore`.

Add `DEBUGY_ENV=development` to your `.env` file. This tells Debugy to save logs locally. Without it, logging is disabled. For production cloud logging, set `DEBUGY_ENV=production` instead.

## Auto Error Capture

When setting up the logger, also register global error and exception handlers for the project's language so that uncaught errors are automatically logged to `.debugy/session.ndjson` with `level: "error"` and the stack trace in `metadata.stack`.

Examples of what to hook into (adapt to the project's language):
- **Node.js**: `process.on('uncaughtException')`, `process.on('unhandledRejection')`
- **Python**: `sys.excepthook`, `threading.excepthook`
- **Go**: `recover()` in a top-level deferred function
- **Browser**: `window.addEventListener('error')`, `window.addEventListener('unhandledrejection')`

Register these handlers once at app startup. Important:
- Do not suppress the original error behavior (e.g. still exit after uncaught exceptions in Node)
- Avoid double-logging errors that are already caught and logged by `debugy.log()`
- Skip framework noise that is not actionable (e.g. HMR, hydration warnings)

## After Install

Once setup is complete, scan the codebase and suggest 3-5 high-value places to add permanent `debugy.log()` calls — API entry points, error handlers, auth flows, database queries, or external service calls. Present them to the user for approval before adding.

## Workflow

When asked to debug with Debugy:
1. Clear `.debugy/session.ndjson` to start a fresh session
2. Add `debugy.log()` calls where runtime visibility is missing — use `level: "info"` for general flow, `level: "error"` for failures
3. Ask the user to run or reproduce the issue
4. Read `.debugy/session.ndjson`, diagnose errors, warnings, timings, and patterns
5. Apply the fix
6. Remove temporary Debugy calls when done (keep any permanent ones the user approved)

## Reading Logs

Read the full session:
```bash
cat .debugy/session.ndjson
```

Errors only:
```bash
grep '"level":"error"' .debugy/session.ndjson
```

Last 50 entries:
```bash
tail -50 .debugy/session.ndjson
```

Search for a term:
```bash
grep '<QUERY>' .debugy/session.ndjson
```

## Clear Logs

Start a fresh session:
```bash
rm -f .debugy/session.ndjson
```

## Housekeeping

- Clear the session file at the start of each new debug workflow
- If the file exceeds ~500 lines mid-session, keep only the latest 200:
  ```bash
  tail -200 .debugy/session.ndjson > .debugy/session.tmp && mv .debugy/session.tmp .debugy/session.ndjson
  ```
- Always inform the user when cleaning up: how many debugy.log() calls were removed, whether the session file was cleared or truncated, and how many lines were kept

## debugy.log() Contract

```ts
debugy.log(file, fn, message, { level?, duration_ms?, metadata? })
```

- Always pass `file`, `fn`, and `message`
- Fire-and-forget; never await in hot paths
- Never log secrets, tokens, credentials, or PII

## Rules

- Ask before adding logs or running code
- Prefer high-signal logs with real values in `metadata`
- Focus on API entry/exit, external calls, decision branches, async boundaries, and error paths
- Remove Debugy calls when debugging is complete

## Cloud Mode

`DEBUGY_ENV` decides where logs go. In development (default), logs save to the local file. When `DEBUGY_ENV=production` (or `staging`, etc.) and `DEBUGY_WRITE_KEY` is set, logs go to the cloud API instead. The agent reads cloud logs via `DEBUGY_AGENT_KEY`.