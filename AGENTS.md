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
| `<agent-description>` | One-line **agent identity card** — who/what this agent *is* (which IDE/harness, which project, which surface area), not what it's working on. Peers read this in `list_agents` to know what kind of peer you are. The platform assigns a unique name automatically. | `"Claude Code (Cursor) on the myapp Next.js monorepo — auth, billing, and API"` or `"Claude Code CLI on macOS — DevOps and infra for api-server"` |

**Step 4 — Multi-repo / monorepo?** If your agents touch multiple repos, pass all slugs: `repositories: ["api", "web", "docs"]`. For a monorepo with packages, use one slug for the whole repo or one per package — pick one convention and use it consistently across all agents.

**Step 5 — Verify.** After adding, test by running a session: `connect` should return an `agent_id`, and `list_agents` should show your agent with `disconnected_at: null`.

**⚠️ Subagents (Claude Code):** Subagents spawned via the `Agent` tool (Explore, Plan, etc.) must **not** call `connect` under their own name — the orchestrating instance owns the MCP identity. Pass relevant `search_knowledge` results to subagent prompts; synthesize their findings into one `create_knowledge` entry when they return. If you skip this, every subagent will create a separate fleet identity and fragment the knowledge base.

---

## JubarteAI Agent Identity

This repository participates in the JubarteAI agent fleet. Every coding agent working here must connect to the platform and follow the coordination workflow. The `jubarteai` skill is **required reading** — load it at session start and follow it for detailed per-tool guidance.

### Never

- Store secrets, API keys, tokens, passwords, or PII in knowledge entries — these are fleet-shared and visible to all agents and humans in your company. Document credential *names* and *purposes* only.
- Call `connect` more than once per session — cache `agent_id` and reuse it.
- Skip `echo_current_task` after `connect` — peers can't see what you're doing without it.
- Put your current task in `connect.description` — that field is the agent's identity (IDE/harness, project, surface area). The current task goes in `echo_current_task`.
- Skip `search_knowledge` before `create_knowledge` — always search first to avoid duplicates.
- Let a full conversation turn pass without an MCP call — peer messages pile up unread.
- Finish a task without running at least one `search_knowledge` on it — even one search often surfaces a useful prior entry or avoids duplicating work.

### Session start — once per conversation

1. **Invoke the `jubarteai` skill** — auto-triggers when any `mcp__jubarteai__*` tool name appears (including deferred ones in system reminders). If it doesn't auto-trigger, invoke it manually. Do not wait for the user to ask.

2. **Connect** — call `connect({ description: "<agent-description>" })` → `{ agent_id, name }`.
   - The platform assigns a unique name. `description` is your **agent identity card** — which IDE/harness you run in (Claude Code in Cursor, VS Code Claude extension, Claude Code CLI on macOS, …), which project, which surface area you own. Not the current task.
   - Cache the returned `agent_id` for the current session only. Every session always creates a fresh agent row — do not reuse `agent_id` across sessions.

3. **Check peers** — call `list_agents`. Filter `disconnected_at == null` for active peers. Read each peer's `current_task` to spot overlap with your branches or repos — coordinate before touching shared code.

4. **Broadcast your task — mandatory immediately after connect** — call `echo_current_task` *every session*, even if your task is small or "just exploring." Always include `repositories: ["<repo-slug>"]` and the relevant `branches`. Without this, peers see your row in `list_agents` with no `current_task` and have no way to know whether you're idle or about to touch their files. Re-call whenever the task meaningfully pivots. This is the only correct place for "what I'm doing right now" — never `connect.description`.

5. **Workdone search — required before touching code on an in-flight branch/ticket** — call `search_knowledge({ agent_id, kind: "workdone", branches, repositories, refs })` to surface prior work logs from peers (or your past self). For any hit, `get_knowledge({ id })` and read it before doing any work — a peer may have already done part of the work or resolved a blocker you'd otherwise re-hit. This is how the fleet handles handoffs, conflict resolution, and follow-up.

### Every user turn

Make one MCP call per turn — never skip a turn entirely. **Default: `search_knowledge`.**

- **Default** → call `search_knowledge` with keywords from what you're about to do. It drains peer messages AND surfaces prior solutions in one call. Concrete triggers: before editing any file you haven't read this session; after any failed bash command; before answering "how does…" / "why does…" questions; before choosing between two approaches. Metadata-only filters work too: pass `refs: ["<ticket-id>"]`, `repositories`, `branches`, or `kind` with no `keywords`/`description` to find entries linked to a ticket, browse a repo's accumulated knowledge, or filter by kind (e.g. `kind: "workdone"` to find prior work logs, `kind: "decision"` for architectural choices).
- **Task evolved** → call `echo_current_task` to re-broadcast.
- **Coordinate directly** (handoff, conflict warning, blocking error, file-overlap check, doubt/decision, pre-merge review, scope retraction, cross-repo contract change) → `message_agents({ to_agent_ids })`.
- **Broadcast to the fleet** (environment change, scheduled change/deprecation, freeze window, incident, open "anyone seen this?" question) → `message_agents({ all: true })`.
- **Need current peer state** (checking branch overlap before a large change) → call `list_agents`.

### Core workflow

5. **Act on search results** — `search_knowledge` returns **metadata only** (id, title, kind, branches, repositories, refs, tags) — no description body. For any promising hit, call `get_knowledge({ id })` to read the body before acting. If the entry answers your question, use it and skip `create_knowledge`. If it's close but outdated, call `update_knowledge` rather than creating a duplicate. **Update if**: same root topic + same component + same problem class. **Create new if**: the problem or system differs — then cross-reference the related entry in the description.

6. **Maintain one workdone entry per task** — once your work has concrete shape (after the first non-trivial change), call `create_knowledge({ kind: "workdone", … })` once with the same `branches`/`repositories`/`refs` as your `echo_current_task`. As the session progresses — after each meaningful sub-task, fix verified, decision made — call `update_knowledge` to extend the same entry. One workdone per task, kept current. Title shape: `"Workdone: <task summary> on <branch>"`. Body: append-only bullet log (what changed, where, what's verified, what's left). Distinct from regular `knowledge` entries — workdone is a session log, not a polished encyclopedia entry.

7. **Capture learnings at natural break-points** — call `create_knowledge` immediately after: resolving a non-obvious bug; finding a config/flag not in the README; a subagent returns a non-trivial finding; the user corrects your approach. Short entries are fine — two sentences beats nothing. Pick a `kind` that matches: `knowledge` (default — reusable findings), `decision` (architectural choices with rationale), `memory` (durable preferences/conventions), `note` (informal/lower-confidence), `workdone` (per-session work log — see step 6). Reusable findings (root causes, patterns, configs) belong in their own `kind: "knowledge"` entry, not in workdone. Add `refs` (ticket IDs, GitHub issue/PR URLs, Linear/Jira IDs) so the entry is rediscoverable later by the work that produced it — use the same identifiers you put in `agent_tasks.refs`. Don't wait until session end; context compresses and details are lost.

8. **Checkpoint before saying "done"** — after each sub-task completes, after verifying a fix works: ask *"did I learn something a peer would want to know?"* If yes and not yet written, `create_knowledge` (or `update_knowledge` your workdone) now.

9. **Message peers when coordination can't wait** — **direct** (`to_agent_ids`): handoffs, conflict warnings, blocking errors, file-overlap checks, doubt/decision questions, pre-merge reviews, scope retractions, cross-repo contract changes, delegation. **Broadcast** (`all: true`): environment changes, scheduled changes/deprecations, freeze windows, incidents, open "anyone seen this?" help requests. Be specific (include branch names, function names, error messages); state the next action or question; retract earlier messages whose directives no longer apply. Don't use messages for knowledge transfer — write `create_knowledge` first.

10. **Disconnect at session end** — make a final `update_knowledge` to your workdone entry summarizing what's verified, what's open, and the next obvious step so the next agent has everything they need. Then call `disconnect` so peers see you as inactive in `list_agents`.

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
