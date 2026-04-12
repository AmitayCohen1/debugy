---
name: debugy-local
description: Local runtime debugging workflow for Debugy.
allowed-tools: Bash(node *) Read Write Edit Glob Grep
---

# Debugy Local

Use Debugy Local when static code reading is not enough and you need runtime evidence from a local reproduction.

## Two local log sources

1. **Server log** — `.debugy/server.log` passively captures all dev server output. Check this first — the error is often already there.
2. **`debugy.log()`** — structured logs you add for targeted visibility. In Debugy Local, logs write to `.debugy/session.ndjson` when `DEBUGY_ENV=development`. If `DEBUGY_ENV` is not set, logging stays disabled.

Always use `debugy.log()`. Never build a custom logger or HTTP client. Adapt the template below to the project's language.

## Logger template

```ts
// lib/debugy.ts
import { appendFileSync, mkdirSync } from "node:fs";

const DEBUGY_ENV = process.env.DEBUGY_ENV ?? "";

function log(file: string, fn: string, message: string, opts: { level?: string; duration_ms?: number; metadata?: Record<string, string | number | boolean | null> } = {}) {
  if (DEBUGY_ENV !== "development") return;

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

  if (typeof window !== "undefined") {
    fetch("/api/debugy", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(entry),
    }).catch(() => {});
    return;
  }

  try {
    mkdirSync(".debugy", { recursive: true });
    appendFileSync(".debugy/session.ndjson", JSON.stringify(entry) + "\n");
  } catch {
    console.warn("debugy-local: failed to write log");
  }
}

export const debugy = { log };
```

If the project uses a different language, implement the same behavior.

**Browser logs:** Add a local API route so client logs reach the same file:

```ts
// app/api/debugy/route.ts
import { appendFileSync, mkdirSync } from "node:fs";

export async function POST(req: Request) {
  const entry = await req.json();
  mkdirSync(".debugy", { recursive: true });
  appendFileSync(".debugy/session.ndjson", JSON.stringify(entry) + "\n");
  return new Response("ok");
}
```

## Setup

1. Add `.debugy/` to `.gitignore`.
2. Create a `dev:debugy` script that pipes server output to `.debugy/server.log`:
   - **Node.js:** `"dev:debugy": "mkdir -p .debugy && FORCE_COLOR=1 npm run dev 2>&1 | tee .debugy/server.log"`
   - **Python:** `mkdir -p .debugy && FORCE_COLOR=1 python manage.py runserver 2>&1 | tee .debugy/server.log`
   - **Go:** `mkdir -p .debugy && FORCE_COLOR=1 go run . 2>&1 | tee .debugy/server.log`
3. Set `DEBUGY_ENV=development` in the app env when you want structured local logs.
4. Suggest 3-5 high-value places for temporary `debugy.log()` calls. Present them to the user before adding them.

## Workflow

1. Clear logs: `rm -f .debugy/session.ndjson .debugy/server.log`
2. Check `.debugy/server.log` first for errors and stack traces
3. If more visibility is needed, add `debugy.log()` calls at targeted spots
4. Ask the user to reproduce the issue locally
5. Read logs, diagnose, fix
6. Remove temporary `debugy.log()` calls when done

## Reading logs

```bash
cat .debugy/server.log
cat .debugy/session.ndjson
grep -i 'error' .debugy/server.log
grep '"level":"error"' .debugy/session.ndjson
```

## `debugy.log()` contract

```
debugy.log(file, fn, message, { level?, duration_ms?, metadata? })
```

- Fire-and-forget; never await in hot paths
- Never log secrets, tokens, credentials, or PII

## Rules

- Ask before adding logs or running code
- Check `server.log` before adding manual instrumentation
- Prefer high-signal logs with real values in `metadata`
- Remove temporary Debugy calls when debugging is complete

## Housekeeping

- Cap `.debugy/server.log` at 500 lines: `tail -500 .debugy/server.log > .debugy/server.tmp && mv .debugy/server.tmp .debugy/server.log`
- Cap `.debugy/session.ndjson` at 200 lines: `tail -200 .debugy/session.ndjson > .debugy/session.tmp && mv .debugy/session.tmp .debugy/session.ndjson`
