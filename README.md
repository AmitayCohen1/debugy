# Debugy Skill

Give your coding agent runtime visibility. Debugy captures your dev server output to a file the agent can read, and lets it add targeted `debugy.log()` calls when it needs deeper insight.

No server, no API keys, no network. Just local files.

## Quick start

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

Copy the contents of `codex/AGENTS.md` into your project's `AGENTS.md` file. If you already have one, add the Debugy section — don't replace existing instructions.

## How it works

1. Paste the skill into your agent
2. Start your dev server with `<your-dev-command> 2>&1 | tee .debugy/server.log`
3. Hit a bug — your agent reads `.debugy/server.log` for errors and stack traces
4. If it needs more detail, it adds `debugy.log()` calls at targeted spots
5. You reproduce the issue
6. Your agent reads the logs, sees what actually happened, and fixes the bug
7. It removes the temporary log calls when done

## Language support

The instructions include a TypeScript example, but the agent adapts to whatever language your project uses — Python, Go, Ruby, etc. The log format and file path are the same.

## Cloud mode

For production logging, check out [debugy.dev](https://www.debugy.dev). Cloud mode sends logs to a hosted API so your agent can debug real-world issues.

## Community

[Join our Discord](https://discord.gg/ZUmepgyY)

## License

MIT
