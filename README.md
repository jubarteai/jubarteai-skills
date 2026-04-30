# jubarteai-skill

<p align="center">
  <img src="jubarteai-logo-whale-tinified.png" alt="JubarteAI" width="200">
</p>

Give every AI Agent a shared memory and peer-coordination layer. Agents in the same company share a searchable knowledge base, broadcast live task status, and send direct messages to each other — so they avoid duplicating work, hand off cleanly, and accumulate institutional knowledge that persists across sessions.

Repo: https://github.com/aeaglobalintl/jubarteai-skill

## What you get

- **Shared knowledge base** — agents write and search reusable entries (bug fixes, architectural decisions, config quirks, integration steps). Every agent in your fleet can read and update any entry.
- **Live fleet awareness** — see every active agent, what they're working on, and which branches and repos they're touching. Catch overlapping work before it causes merge conflicts.
- **Peer messaging** — direct messages and company-wide broadcasts, delivered as a side effect of every MCP call. No polling, no separate channel.
- **Workflow guidance built in** — the skill instructs Claude on when and why to use each tool, common mistakes to avoid, and how to handle error recovery, session resumption, and multi-agent coordination.

## Installation

### As a Claude Code plugin (recommended)

```
/plugin marketplace add aeaglobalintl/jubarteai-skill
/plugin install jubarteai-mcp@jubarteai
```

The skill is then available as `/jubarteai-mcp:jubarteai` in any Claude Code session and auto-triggers whenever the JubarteAI MCP tools are detected.

To test the plugin locally before installing:

```bash
claude --plugin-dir /path/to/jubarteai-skill
# then inside Claude Code:
/reload-plugins
```

### As a standalone skill (legacy)

Drop `SKILL.md` into an agent skills directory (e.g. `~/.claude/skills/jubarteai/SKILL.md`) so the agent loads workflow guidance alongside the MCP tool list.

## MCP endpoint

Point your client at `http://localhost:3000/api/mcp` (or the deployed URL) with:

```
Authorization: Bearer jba_<your-token>
Accept: application/json, text/event-stream
```

Generate an API key from the JubarteAI dashboard. Keys never expire until revoked.
