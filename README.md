# Debugy

A dead-simple log API for your coding agent. Any service or pipeline POSTs its logs; your agent fetches them back. One key, no SDK, no infra — just an env var and HTTP.

## How it works

1. **Write** — any service POSTs a log to `https://www.debugy.dev/api/logs` with `DEBUGY_KEY`.
2. **Read** — your agent fetches logs from the same endpoint with the same key, filtered by service, level, or text.

That's it. One bucket, every pipeline writes to it, the agent reads it.

## Quick start

1. Create a free account at https://www.debugy.dev/sign-up to get your `DEBUGY_KEY`.
2. Set `DEBUGY_KEY=dbg_...` in any service that should log, and keep it available to your agent / shell.
3. Give your agent the Debugy instructions for your tool (below).

### Write a log (any language)

```bash
curl -X POST "https://www.debugy.dev/api/logs" \
  -H "Authorization: Bearer $DEBUGY_KEY" \
  -H "Content-Type: application/json" \
  -d '{"service":"checkout-api","level":"error","message":"payment declined","metadata":{"code":402}}'
```

- `service` (required) — which pipeline/service the log came from
- `message` (required) — the log line
- `level` (optional) — `debug` | `info` | `warn` | `error` (default `info`)
- `metadata` (optional) — any JSON object

### Read logs (the agent)

```bash
curl -s "https://www.debugy.dev/api/logs?service=checkout-api&level=error&limit=100" \
  -H "Authorization: Bearer $DEBUGY_KEY"
```

Filters: `service`, `level`, `search`, `limit`, `offset`.

## Install the agent instructions

Pick your agent and copy the file into your project:

### Claude Code

```bash
mkdir -p .claude/skills/debugy
cp claude/SKILL.md .claude/skills/debugy/SKILL.md
```

### Cursor

```bash
mkdir -p .cursor/rules
cp cursor/debugy.mdc .cursor/rules/debugy.mdc
```

### Codex

Copy the contents of `codex/AGENTS.md` into your project's `AGENTS.md`. If you already have one, add the `## Debugy` section — don't replace existing instructions.

## Safety

- Never log secrets or PII — passwords, tokens, keys, session IDs, emails. Log shapes, not values.
- Keep `DEBUGY_KEY` server-side; route browser logs through a server endpoint. Never put the key in client code.

## Community

[Join our Discord](https://discord.gg/ZUmepgyY)

## License

MIT
