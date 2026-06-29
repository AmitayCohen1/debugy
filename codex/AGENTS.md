## Debugy

This section is intended to live inside your repository AGENTS.md alongside the rest of your project guidance. Merge it instead of replacing unrelated instructions.

Debugy is a dead-simple log API. Any service or pipeline POSTs logs; you fetch them back. One key for both, no SDK, no setup — just an env var and HTTP.

### Key

`DEBUGY_KEY` (`dbg_...`) — one key. Services write with it, the agent reads with it. Get it free at https://www.debugy.dev/sign-up. Keep it server-side; never expose it to a browser.

### Writing logs

One HTTP POST from anywhere — any language, any service:

```
POST https://www.debugy.dev/api/logs
Authorization: Bearer <DEBUGY_KEY>
Content-Type: application/json

{"service":"checkout-api","level":"error","message":"payment declined","metadata":{"code":402}}
```

Fields:
- `service` (required) — which pipeline/service the log came from. This is how you tell logs apart later.
- `message` (required) — the log line.
- `level` (optional) — `debug` | `info` | `warn` | `error`. Defaults to `info`.
- `metadata` (optional) — any JSON object of extra context.

The server stamps `timestamp` and an `id`.

#### Optional helper (JS/TS)

If a project logs from many spots, drop in a one-function helper instead of repeating the fetch:

```ts
// lib/debugy.ts
export function debugy(service: string, message: string, opts: { level?: string; metadata?: Record<string, unknown> } = {}) {
  const key = process.env.DEBUGY_KEY;
  if (!key) return;
  fetch("https://www.debugy.dev/api/logs", {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${key}` },
    body: JSON.stringify({ service, level: opts.level ?? "info", message, ...(opts.metadata && { metadata: opts.metadata }) }),
  }).catch(() => {});
}
```

Keep the key server-side. For browser logs, route them through a server endpoint that calls this — never put the key in client code. Same idea in any other language: a plain HTTP POST is enough.

### Reading logs (say `debugy`)

When the user says `debugy`, fetch recent logs with `DEBUGY_KEY` (check `.env`, `.env.local`, or the shell env). Filter by `service`, `level`, `search`, `limit`, and `offset`:

```bash
# everything recent
curl -s "https://www.debugy.dev/api/logs?limit=100" -H "Authorization: Bearer <DEBUGY_KEY>"
# one pipeline
curl -s "https://www.debugy.dev/api/logs?service=checkout-api&limit=100" -H "Authorization: Bearer <DEBUGY_KEY>"
# errors only
curl -s "https://www.debugy.dev/api/logs?level=error&limit=100" -H "Authorization: Bearer <DEBUGY_KEY>"
# full-text search
curl -s "https://www.debugy.dev/api/logs?search=payment&limit=50" -H "Authorization: Bearer <DEBUGY_KEY>"
# clear logs
curl -X DELETE "https://www.debugy.dev/api/logs" -H "Authorization: Bearer <DEBUGY_KEY>"
```

If the response includes a `notice` field, surface it to the user — it signals limits (e.g. quota) that affect what you can see. If no key is found, tell the user to grab one at https://www.debugy.dev/sign-up.

### Rules

- `service` + `message` are all you need; add `metadata` for real values worth inspecting.
- Never log secrets or PII — passwords, tokens, keys, session IDs, emails. Log shapes, not values (e.g. `userId length: 36`).
- Keep `DEBUGY_KEY` server-side, never in the browser.
