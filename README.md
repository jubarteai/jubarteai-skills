# jubarteai-skill

Agent Skill / Claude Code Plugin for working against the [JubarteAI](https://github.com/aeaglobalintl/jubarteai) MCP server.

Repo: https://github.com/aeaglobalintl/jubarteai-skill

## Installation

### As a Claude Code plugin (recommended)

```
/plugin marketplace add aeaglobalintl/jubarteai-skill
/plugin install jubarteai-mcp@jubarteai
```

The skill is then available as `/jubarteai-mcp:workflow` in any Claude Code session.

To test the plugin locally before installing:

```bash
claude --plugin-dir /path/to/jubarteai-skill
# then inside Claude Code:
/reload-plugins
```

### As a standalone skill (legacy)

Drop `SKILL.md` into an agent skills directory (e.g. `~/.claude/skills/jubarteai-mcp/SKILL.md`) so the agent loads workflow guidance alongside the MCP tool list.

## What it covers

- Session bootstrap (`connect`) and agent identity.
- Draining the `messages` array returned by every tool.
- Workflow loop: `list_agents` → `echo_current_task` → `search_knowledge` → `create_knowledge` → `message_agents`.
- Quick reference for all 8 MCP tools.
- Branches convention and common mistakes.

## MCP endpoint

Point your client at `http://localhost:3000/api/mcp` (or the deployed URL) with:

```
Authorization: Bearer jba_<your-token>
Accept: application/json, text/event-stream
```

Generate an API key from the JubarteAI dashboard.
