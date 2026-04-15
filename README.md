# Debugy Local

Give your coding agent runtime visibility during local development. Debugy Local captures your dev server output to a file the agent can read, and lets it add targeted `debugy.log()` calls when it needs deeper insight.

No server, no API keys, no network. Just local files on your machine.

## Quick start

Pick your agent and copy the file into your project:

### Claude Code

```bash
mkdir -p .claude/skills/debugy-local
cp claude/SKILL.md .claude/skills/debugy-local/SKILL.md
```

### Cursor

```bash
mkdir -p .cursor/rules
cp cursor/debugy-local.mdc .cursor/rules/debugy-local.mdc
```

### Codex

Copy the contents of `codex/AGENTS.md` into your project's `AGENTS.md` file. If you already have one, add the `## Debugy Local` section — don't replace existing instructions.

## How it works

1. Paste the Debugy Local skill into your agent
2. Start your dev server with `<your-dev-command> 2>&1 | tee .debugy/server.log`
3. The agent sets up a global error hook and adds `debugy.log()` calls where they provide runtime visibility
4. Hit a bug — your agent reads `.debugy/server.log` for errors and stack traces
5. If it needs more detail, it adds more targeted `debugy.log()` calls
5. You reproduce the issue
6. Your agent reads the logs, sees what actually happened, and fixes the bug
7. It removes the temporary log calls when done

## Language support

The instructions include a TypeScript example, but the agent adapts to whatever language your project uses — Python, Go, Ruby, etc. The log format and file path are the same.

## Housekeeping

Log files can grow during long sessions. The skill instructions tell your agent to manage this automatically, but you can also do it manually:

```bash
# Cap server.log at the last 500 lines
tail -500 .debugy/server.log > .debugy/server.tmp && mv .debugy/server.tmp .debugy/server.log

# Keep only the latest 200 structured log entries
tail -200 .debugy/session.ndjson > .debugy/session.tmp && mv .debugy/session.tmp .debugy/session.ndjson

# Start fresh
rm -f .debugy/session.ndjson .debugy/server.log
```

## Cloud mode

For staging and production logging, check out [debugy.dev](https://www.debugy.dev). Debugy Cloud is a separate hosted skill for remote incidents.

## Community

[Join our Discord](https://discord.gg/ZUmepgyY)

## License

MIT
