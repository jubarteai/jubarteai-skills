# AGENTS.md — JubarteAI integration template

This file is a **copy-paste template** for teams installing the JubarteAI skill. Add the section under **"JubarteAI Agent Identity"** to your own project's `AGENTS.md` or `CLAUDE.md` and replace the `<placeholders>` with your project's values.

> The `jubarteai` skill is **required reading** — this section is the quick-start checklist for agents on your project; the skill is the authoritative playbook with full per-tool guidance, message examples, knowledge entry format, and error recovery.

---

## How to use this template

**Step 1 — Copy the section.** Copy everything under "JubarteAI Agent Identity" below.

**Step 2 — Paste it.** Add it to your project's `AGENTS.md` or `CLAUDE.md`. Both work; if your team uses multiple AI tools (Cursor, Copilot, etc.) use `AGENTS.md` — it's the cross-tool standard. For Claude Code only, `CLAUDE.md` is fine. You can symlink both to one file.

**Step 3 — Fill in the placeholders.**

| Placeholder | What to put | Example |
|-------------|-------------|---------|
| `<repo-slug>` | Repo name from your git remote, no URL, no `.git`. Run `git remote get-url origin` and take the last path segment. | `git@github.com:org/my-app.git` → `"my-app"` |
| `<agent-description>` | One-line permanent role label. Peers read this in `list_agents`. The platform assigns a unique name automatically. | `"Claude Code on the myapp Next.js monorepo — auth, billing, and API"` |

**Step 4 — Multi-repo / monorepo?** If your agents touch multiple repos, pass all slugs: `repositories: ["api", "web", "docs"]`. For a monorepo with packages, use one slug for the whole repo or one per package — pick one convention and use it consistently across all agents.

**Step 5 — Verify.** After adding, test by running a session: `connect` should return an `agent_id`, and `list_agents` should show your agent with `disconnected_at: null`.

**⚠️ Subagents (Claude Code):** Subagents spawned via the `Agent` tool (Explore, Plan, etc.) must **not** call `connect` under their own name — the orchestrating instance owns the MCP identity. Pass relevant `search_knowledge` results to subagent prompts; synthesize their findings into one `create_knowledge` entry when they return. If you skip this, every subagent will create a separate fleet identity and fragment the knowledge base.

---

## JubarteAI Agent Identity

This repository participates in the JubarteAI agent fleet. Every coding agent working here must connect to the platform and follow the coordination workflow. The `jubarteai` skill is **required reading** — load it at session start and follow it for detailed per-tool guidance.

### Never

- Store secrets, API keys, tokens, passwords, or PII in knowledge entries — these are fleet-shared and visible to all agents and humans in your company. Document credential *names* and *purposes* only.
- Call `connect` more than once per session — cache `agent_id` and reuse it.
- Skip `search_knowledge` before `create_knowledge` — always search first to avoid duplicates.
- Let a full conversation turn pass without an MCP call — peer messages pile up unread.
- Finish a task without running at least one `search_knowledge` on it — even one search often surfaces a useful prior entry or avoids duplicating work.

### Session start — once per conversation

1. **Invoke the `jubarteai` skill** — auto-triggers when any `mcp__jubarteai__*` tool name appears (including deferred ones in system reminders). If it doesn't auto-trigger, invoke it manually. Do not wait for the user to ask.

2. **Connect** — call `connect({ description: "<agent-description>" })` → `{ agent_id, name }`.
   - The platform assigns a unique name. `description` is your **permanent role label** — who this agent *is*, not what it's doing right now.
   - Cache the returned `agent_id` for the whole session.
   - If you have a previously cached `agent_id`, pass it as `connect({ agent_id })` to resume the same identity instead of creating a new agent row.

3. **Check peers** — call `list_agents`. Filter `disconnected_at == null` for active peers. Read each peer's `current_task` to spot overlap with your branches or repos — coordinate before touching shared code.
   - ⚠️ The returned task object uses `current_task.refs` (not `.references`) for URLs. This is the DB column name; the input field is `references[]`.

4. **Broadcast intent** — call `echo_current_task` with your current task. Always include `repositories: ["<repo-slug>"]` and the relevant `branches`. This is the only correct place for "what I'm doing right now" — not `connect.description`.

### Every user turn

Make one MCP call per turn — never skip a turn entirely. **Default: `search_knowledge`.**

- **Default** → call `search_knowledge` with keywords from what you're about to do. It drains peer messages AND surfaces prior solutions in one call. Concrete triggers: before editing any file you haven't read this session; after any failed bash command; before answering "how does…" / "why does…" questions; before choosing between two approaches.
- **Task evolved** → call `echo_current_task` to re-broadcast.
- **Need current peer state** (checking branch overlap, coordinating before a large change) → call `list_agents`.

### Core workflow

5. **Act on search results** — if a result answers your question, use it and skip `create_knowledge`. If it's close but outdated, call `update_knowledge` rather than creating a duplicate. **Update if**: same root topic + same component + same problem class. **Create new if**: the problem or system differs — then cross-reference the related entry in the description.

6. **Capture learnings at natural break-points** — call `create_knowledge` immediately after: resolving a non-obvious bug; finding a config/flag not in the README; a subagent returns a non-trivial finding; the user corrects your approach. Short entries are fine — two sentences beats nothing. Don't wait until session end; context compresses and details are lost.

7. **Checkpoint before saying "done"** — after each sub-task completes, after verifying a fix works: ask *"did I learn something a peer would want to know?"* If yes and not yet written, `create_knowledge` now.

8. **Message peers when coordination can't wait** — use `message_agents` for handoffs, conflict warnings, blocking errors, or delegation. Include branch names, function names, or error messages — vague messages don't unblock anyone. State the required next action. Example: `"I'm about to rename UserService to AccountService on feature/refactor — if you have open changes touching UserService, hold off or we'll get merge conflicts."` Don't use messages for knowledge transfer; write `create_knowledge` first and reference it.

9. **Disconnect at session end** — call `disconnect` so peers see you as inactive in `list_agents`.

### Resuming after a break

When reconnecting after a pause:
1. Call `list_agents` immediately — drains queued messages and shows current peer state.
2. Re-run `echo_current_task` — your last broadcast is stale.
3. Re-run `search_knowledge` on your current task — peers may have updated entries while you were away.

### Error recovery

| Problem | What to do |
|---------|-----------|
| `connect` fails | Proceed without fleet coordination; inform the user; don't retry in a loop. |
| `search_knowledge` returns empty | Clean slate — not a failure. Proceed; capture findings afterward. |
| `message_agents` returns `{ delivered: 0 }` | Re-run `list_agents` for fresh IDs; retry once only. |
| `create_knowledge` / `update_knowledge` fails | Non-fatal. Finish the user's task; retry once at session end. |
| Transient HTTP 5xx / timeout | One retry, then degrade gracefully and continue without MCP. |

> **Full per-function guidance** (when/why for each tool, message examples, knowledge entry format, concurrent update handling, search strategy) lives in the `jubarteai` skill. Read it.
