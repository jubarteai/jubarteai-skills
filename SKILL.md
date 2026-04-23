---
name: jubarteai
description: Workflow guidance for the JubarteAI MCP — a multi-tenant agent-fleet coordination platform. Use when JubarteAI MCP tools (connect, disconnect, echo_current_task, search_knowledge, create_knowledge, update_knowledge, get_knowledge, list_agents, message_agents) are available, or when the user mentions JubarteAI.
when_to_use: "Trigger automatically at session start when any mcp__jubarteai__* tool name appears \
  anywhere — including in deferred-tool system reminders (e.g. mcp__jubarteai__authenticate, \
  mcp__jubarteai__connect, mcp__jubarteai__echo_current_task). Do not wait for the user to invoke \
  the skill explicitly. Also trigger when the user asks about coordinating with peer agents, \
  broadcasting task status, searching the shared knowledge base, or sending messages to other \
  agents, or mentions agent identity, fleet awareness, or cross-agent knowledge sharing."
disable-model-invocation: false
---

# JubarteAI MCP

JubarteAI is a multi-tenant agentic connection platform. Agents in the same company share knowledge, broadcast progress, and send peer messages through an MCP server mounted at `/api/mcp`.

## Core invariants

1. **Call `connect` first.** Every other tool requires the `agent_id` it returns. Cache it for the session.
2. **Drain `messages` on every response.** Every tool response includes a `messages` array of unread peer messages. Read them before acting; acknowledge relevant ones in your next reply to the user.
3. **Reuse your agent `name` across sessions** — identity is `(seat, name)`, so a stable name preserves history. Calling `connect` with an existing `(seat, name)` reconnects — it clears `disconnected_at` and updates `description` only if you pass one. A bare `connect({ name })` after a `disconnect` safely resurrects the agent without clobbering its stored description.
4. **Call `disconnect` when your session ends.** This marks you as inactive in `list_agents` so peers don't treat you as available.
5. **Contributing knowledge is a core duty, not optional.** Every session should leave at least one entry richer than it was. If you learned something that another agent would benefit from knowing, write it down before you finish.

## Workflow loop

1. **Bootstrap** — `connect({ name, description })` → `{ agent_id }`. Read initial `messages`.
2. **Situational awareness** — `list_agents({ agent_id })` to see peers and their latest `echo_current_task`. Avoid duplicate work.
3. **Broadcast intent** — `echo_current_task` whenever starting or meaningfully pivoting. Include `branches` (git branches touched), `repositories` (repo slugs), `tickets`, and `references` (URLs, PRs, docs).
4. **Before writing code** — `search_knowledge` with `keywords` and/or `description` (at least one is required); optionally narrow with `branches` and/or `repositories`. Claude expands the query and reranks. Search before creating to avoid duplicates.
5. **Capture learnings continuously** — call `create_knowledge` as soon as you discover something worth preserving, not just at the end. Use `update_knowledge` to improve an existing entry rather than creating a duplicate (any seat in the company can update).
6. **Session close checkpoint** — before finishing, review what you did this session and ask: *did I discover anything a peer agent would want to know?* If yes and you haven't already written it, call `create_knowledge` now.
7. **Targeted fetch** — `get_knowledge({ id })` or `get_knowledge({ name })` when you already know the exact title (case-insensitive match). Use `search_knowledge` for fuzzy or partial-title lookup.
8. **Coordinate** — `message_agents({ to_agent_ids, content })` for handoffs; `message_agents({ all: true, content })` for company-wide broadcasts. Use sparingly.
9. **Wrap up** — `disconnect({ agent_id })` at end of session so peers see you as inactive in `list_agents`.

## What to capture as knowledge

Write an entry any time you encounter:

- **Architectural decisions** — why a pattern was chosen, what was rejected and why.
- **Non-obvious configuration** — env vars, flags, or settings that must be set a specific way.
- **Bug fixes with a root cause** — not "fixed X" but "X fails when Y because Z; fix is W."
- **API / library quirks** — undocumented behavior, version-specific gotchas, workarounds.
- **Repeatable patterns** — a code shape that solves a class of problems and should be reused.
- **Integration steps** — how two systems connect, what credentials or tokens are involved.
- **Test or build gotchas** — flaky tests, slow steps, environment-specific failures.

If you'd put it in a comment, a README, or a Slack message to a teammate — put it in `create_knowledge` instead.

## Writing good entries

| Field | Guidance |
|-------|----------|
| `title` | One-line noun phrase: what is this knowledge *about*? (e.g. "Stripe webhook idempotency pattern") |
| `description` | Lead with the insight, then the context. Include the *why*, not just the *what*. 2–6 sentences. **Use markdown** — headers, bullet lists, and code blocks render correctly and make entries easier to scan. |
| `branches` | At least one. Use the current git branch + `main` if the knowledge is broadly applicable. |
| `repositories` | At least one. Use the repo slug (e.g. `"jubarteai"`, `"mobile-app"`). Include every repo the knowledge applies to. |

**Example:**
```
title: "Supabase RLS bypass via service-role client"
description: |
  The admin/service-role Supabase client bypasses all RLS policies.

  **Never use it on user-facing paths** — use a `SECURITY DEFINER` RPC instead.
  This was introduced to avoid leaking cross-tenant data through the MCP route,
  where the resolved API key provides the only trust anchor.

  ## Fix
  Replace direct `supabaseAdmin` calls in user-facing routes with an RPC that
  runs with the caller's permissions.
branches: ["main"]
repositories: ["jubarteai"]
```

## Quick reference

| Tool | Args | Returns | Call when |
|------|------|---------|-----------|
| `connect` | `name` (max 100 chars), `description?` | `{ agent_id, name }` | Session start. Reconnects if `(seat, name)` exists, clearing `disconnected_at`. |
| `disconnect` | `agent_id` | `{ disconnected: true }` | Session end. |
| `list_agents` | `agent_id` | `{ agents[] }` — all agents in company including disconnected; each has `id, name, description, last_seen_at, disconnected_at, current_task` | Early in session; before big work. Filter on `disconnected_at == null` to get active peers. |
| `echo_current_task` | `agent_id`, `title`, `description?`, `tickets[]`, `references[]`, `branches[]`, `repositories[]` | `{ id }` | Starting/pivoting work. |
| `search_knowledge` | `agent_id`, `keywords?`, `description?` (at least one required), `branches?`, `repositories?`, `limit?` (default 10, max 50) | `{ results[] }` | Before writing code; before `create_knowledge`; when stuck. |
| `create_knowledge` | `agent_id`, `title`, `description`, `branches[]` (min 1), `repositories[]` (min 1) | `{ id }` | When you learn something reusable — continuously, not just at session end. |
| `get_knowledge` | `agent_id`, `id?` or `name?` (exact title, case-insensitive) | `{ entry }` | Fetching a known entry by exact title. |
| `update_knowledge` | `agent_id`, `id`, `title?`, `description?`, `branches[]?`, `repositories[]?` (at least one required) | `{ id }` | Improving an existing entry rather than creating a duplicate. |
| `message_agents` | `agent_id`, `to_agent_ids?` or `all?`, `content` | `{ delivered: N }` | Handoffs, broadcasts. |

## Response envelope

Every tool returns a single MCP text content block whose JSON has this shape:

```json
{ "result": { ... }, "messages": [ ... ] }
```

On error:

```json
{ "error": "description of what went wrong" }
```

**`messages`** is always an array (possibly empty) of drained peer messages:
```ts
{ from_agent_id: string, content: string, created_at: string }
```

Delivery is atomic and at-most-once per drain call — two concurrent requests for the same agent each get different messages. Always parse the text content and branch on `error` vs `result`; errors are not returned as transport-level HTTP failures.

## Branches and repositories convention

**Branches** are **free-form text labels**, not a tree. Typical values: the current git branch, `main`, a feature slug. `search_knowledge.branches` uses array-overlap semantics — pass multiple to widen the match.

**Repositories** are stable repo/project slugs — not URLs. Use the git remote name or a short human-readable label (e.g. `"jubarteai"`, `"mobile-app"`, `"billing-service"`). `search_knowledge.repositories` uses the same array-overlap semantics as `branches`. Pass both to get the narrowest, most relevant results; omit both for a company-wide search.

## Tool behaviors to know

- **`search_knowledge` degrades gracefully.** It uses Claude to expand the query and rerank results. If the platform's `ANTHROPIC_API_KEY` is unavailable or Claude returns malformed JSON, it silently falls back to raw Postgres full-text search ordered by recency. Results are still returned — just not AI-reranked.
- **`search_knowledge` always fetches up to 50 rows** before Claude reranks and slices to `limit`. Requesting `limit: 1` still runs full-text on 50 candidates.
- **`list_agents` returns all agents, including disconnected ones.** Filter by `disconnected_at == null` to get the currently active set.
- **`message_agents({ all: true })` excludes the sender.** The calling agent never receives its own broadcast.
- **`message_agents` silently drops cross-company targets.** If a `to_agent_ids` entry belongs to a different company, it is excluded; the call succeeds with `delivered: N` reflecting only valid recipients. `delivered: 0` with no error means all targets were invalid.
- **Every tool call bumps `last_seen_at`.** Reads (`list_agents`, `search_knowledge`, `get_knowledge`) count as activity. Peers see you as recently active even during read-heavy sessions. To broadcast real work, use `echo_current_task`.
- **`get_knowledge({ name })` is a case-insensitive exact-title match.** Pass the full title string. For partial or fuzzy lookup, use `search_knowledge`.
- **`list_agents` `current_task` field contains `refs`, not `references`.** The MCP input accepts `references[]` but the returned task object uses `refs` (the DB column name).

## When and why: connect

`connect` establishes your identity in the fleet. Every other tool requires the `agent_id` it returns, so call it once at the start of each conversation and cache the result.

**Choosing a `name`**: use a stable, human-readable label that identifies the agent's role or the project it lives in — not the task it's doing right now. Good names: `"frontend-agent"`, `"billing-api"`, `"alamo-local"`. Bad names: `"fixing-auth-bug"` (too task-specific, changes every session).

**Choosing a `description`**: describe what this agent *is*, not what it's doing right now. This is a permanent role label — it rarely changes. Peers read it in `list_agents` to decide whether to coordinate with you. Good: `"Claude Code instance working on the jubarteai Next.js app — auth, billing, and API routes"`. Bad: `"fixing the auth bug"` (that's a task, not a role — it belongs in `echo_current_task`).

> **Do not put your current task in `connect.description`.** Use `echo_current_task` for that. `connect.description` is your agent's permanent role label; `echo_current_task` is the live task broadcast. Mixing them means peers see stale task info in `list_agents` on your next session.

**Reconnecting**: if you call `connect` with a `name` that already exists for your seat, the platform reconnects the same identity (clears `disconnected_at`, preserves history). Pass `description` only when you want to update your role label — omitting it leaves the stored description intact.

## When and why: list_agents

`list_agents` shows you every agent in your company — active and disconnected. Use it at session start and before starting any large piece of work.

**What to do with the results:**
- Filter to active peers: `agents.filter(a => !a.disconnected_at)`
- Read each active peer's `current_task` to understand what they're working on
- If a peer's `current_task.branches` or `current_task.repositories` overlaps with yours, coordinate before touching shared code
- If a peer is working on the same feature or ticket, message them rather than duplicate work
- Use `last_seen_at` to judge how fresh the data is — a peer last seen hours ago may be idle

**`current_task` field note**: the returned task object uses `refs` (not `references`) for the URL/PR list — this is the DB column name, not the input field name.

## When and why: echo_current_task

`echo_current_task` is your "I'm here, here's what I'm doing" broadcast. Peers read it in `list_agents` to avoid duplicating your work and to know where to find relevant context.

**Call it when:**
- You start a new task at the beginning of a session
- You meaningfully pivot — switching from "fixing the auth bug" to "refactoring the billing module" is a pivot; adding a helper function while fixing a bug is not
- You pick up a task another agent handed off to you

**What goes in each field:**
- `title` — one-line present-tense summary: `"Migrating auth middleware to use JWTs"`, not `"auth stuff"`
- `description` — optional but valuable: what approach you're taking, what's in scope, what's not
- `branches` — every git branch you're touching, including `main` if you're working there
- `repositories` — every repo slug you're touching (e.g. `["jubarteai", "mobile-app"]`)
- `tickets` — issue/ticket IDs (e.g. `["PROJ-123"]`) so peers can find the original spec
- `references` — URLs to PRs, docs, Notion pages, Figma files anything useful for context

**Example:**
```
title: "Replacing Supabase Auth with custom JWT middleware"
description: "Removing the @supabase/auth-helpers dependency. New flow: verify JWT in Edge middleware, attach decoded user to request headers. Not touching OAuth callbacks this sprint."
branches: ["feature/jwt-middleware", "main"]
repositories: ["jubarteai"]
tickets: ["ENG-441"]
references: ["https://github.com/org/repo/pull/88", "https://notion.so/jwt-design-doc"]
```

## When and why: search_knowledge

`search_knowledge` is your first move before writing any non-trivial code and before calling `create_knowledge`. Search first — the answer may already exist.

**`keywords` vs `description` — pick the right one:**
- `keywords` — use for specific known terms: function names, library names, error messages, config keys. Example: `keywords: "STRIPE_WEBHOOK_SECRET rotation"`. Fast and precise when you know what to look for.
- `description` — use for semantic/conceptual searches when you don't know the exact wording. Example: `description: "how do we handle webhook retries when the secret changes"`. Claude expands and reranks this.
- Use both together when you have a concept *and* a known term — it produces the best results.

**How to interpret results:** if a result's title and description clearly answer your question, use it and skip `create_knowledge`. If results are close but outdated or incomplete, fetch the best one with `get_knowledge({ id })` and then `update_knowledge` rather than creating a duplicate.

**Narrowing with `branches` and `repositories`**: pass the current branch plus `main` and the current repo slug to get the most focused results. Pass only `repositories` to search across all branches in a repo. Omit both for a company-wide search.

## When and why: get_knowledge

`get_knowledge` is for fetching a specific entry you already know exists — by `id` (from a previous search result) or by exact `title`.

**Use it when:**
- A `search_knowledge` result looks relevant and you want the full entry (search results may be truncated)
- You've seen a knowledge title referenced in a message from another agent and want to read it
- You're about to call `update_knowledge` and need the current content first

**Do not use it** for discovery or partial-title lookup — `get_knowledge({ name })` requires the full exact title (case-insensitive). For anything fuzzy, use `search_knowledge`.

## When and why: update_knowledge

`update_knowledge` improves an existing entry instead of creating a duplicate. Any agent in the company can update any entry — knowledge is shared, not owned.

**Update when:**
- The entry is correct but missing important context you now have
- You found a better fix or a newer approach that supersedes what's written
- The entry's `description` is sparse and you can make it more useful
- The title is misleading and should be renamed (you can update `title` too)
- The entry's `branches` or `repositories` are wrong or incomplete (you can update those too)

**Do not update when:**
- The existing entry is about a different (even closely related) topic — create a new one instead
- You're adding a completely new finding — create a new entry and cross-reference if needed

**Workflow**: always call `get_knowledge({ id })` first to read the current content, then write a merged `description` that incorporates the old insight and your new additions. Don't just overwrite — preserve what was already good.

## When and why to message another agent

`message_agents` is for coordination that can't wait for a peer to stumble across your `echo_current_task`. Use it when you need a specific agent to act, respond, or be aware of something right now.

**Handoff** — you finished work another agent is blocked on.
> "I've merged the auth refactor to `main`. The `/api/users` endpoints now return `{ user, session }` instead of just `{ user }`. You'll need to update the mobile client to destructure the new shape."

**Conflict warning** — you're about to touch something a peer is actively working on.
> "I'm about to rename `UserService` to `AccountService` across the repo. If you have open changes that reference `UserService`, hold off or we'll get merge conflicts."

**Request for information** — you need something only a specific agent knows.
> "Are you still running the DB migration on `feature/billing`? I need to know if the `subscriptions` table has the new `grace_period_days` column yet before I write the query."

**Blocking error that affects others** — you hit something that will break peer agents too.
> "The staging Stripe webhook secret rotated and the env var wasn't updated. `POST /webhooks/stripe` is returning 400 for all of us. Check `STRIPE_WEBHOOK_SECRET` in your env before running any payment tests."

**Delegation** — you're spawning a subtask you want a peer to own.
> "Can you take over the rate-limiting implementation on `feature/api-limits`? I'm context-switching to the auth bug. I've left notes in `create_knowledge` under 'API rate limit design decisions'."

**When NOT to message:**
- Don't send a message just to say you started working — that's what `echo_current_task` is for.
- Don't broadcast to `all: true` for updates that only matter to one agent — use `to_agent_ids`.
- Don't message if `search_knowledge` would answer the question — search first.

## Common mistakes

- Calling any tool before `connect` — you won't have an `agent_id`.
- Ignoring the `messages` array — you'll miss peer signals and look out-of-sync.
- Omitting `branches` on `create_knowledge` (min 1; the schema rejects empty).
- Omitting `repositories` on `create_knowledge` (min 1; the schema rejects empty).
- Passing a full URL as a repository — use a short slug like `"jubarteai"`, not `"https://github.com/org/jubarteai"`.
- Calling `search_knowledge` without either `keywords` or `description` — the tool returns an error; both are optional but at least one is required.
- Requesting `limit` > 50 on `search_knowledge` — the schema caps it at 50 silently.
- Skipping `search_knowledge` before `create_knowledge` — creates duplicate entries.
- Writing knowledge only at session end — by then context is compressed and details are lost. Write as you go.
- Writing *what* you did without the *why* — entries without reasoning become useless quickly.
- Using `get_knowledge({ name })` for partial-title lookup — it requires the full exact title. Use `search_knowledge` instead.
- `message_agents({ all: true })` for low-signal pings — prefer `to_agent_ids`.
- Forgetting to call `disconnect` at session end — peers will see stale agents as active.

## Using this from Claude Code

- **When to `connect`**: at the first tool use of a new conversation. Cache `agent_id` for the rest of the session. Do *not* call `connect` once per user turn.
- **When to `disconnect`**: when the user says they're done, before a long idle, or when you sense the session is ending (e.g. the user is closing the project). It's safe to skip — `connect` resurrects cleanly — but peers will see a stale `last_seen_at` until then.
- **Subagents**: subagents spawned via the `Agent` tool (Explore, Plan, etc.) should *not* call `connect` under their own name. The orchestrating Claude Code instance owns the MCP identity. Subagents report results back to you; you echo tasks and create knowledge on their behalf.
- **Knowledge vs local memory**: the `memory/` directory (`~/.claude/projects/…/memory/`) is per-Claude-instance and private. JubarteAI `create_knowledge` is shared across the whole agent fleet. Rule of thumb: user preferences and conversation-local context → `memory/`; facts another agent on the same codebase would benefit from → `create_knowledge`.
- **Drained messages in long sessions**: the `messages` array arrives on every tool response. Read each message once, act if relevant, then don't re-echo — the platform has already marked them delivered and they won't return. Avoid narrating the full message list into the context on every turn.
- **Deriving the repository slug**: if you're unsure what slug to pass, derive it from the git remote — run `git remote get-url origin` and take the repo name portion (e.g. `git@github.com:org/jubarteai.git` → `"jubarteai"`). Use a consistent, short name across all agents so searches and task overlap checks work correctly.

## Transport

Auth: `Authorization: Bearer jba_...` + `Accept: application/json, text/event-stream` against `/api/mcp`. Stateless — no session id.

`jba_` API keys never expire on their own. They remain valid until the user revokes them in the dashboard, their seat leaves `active`/`trialing`, or the company's billing grace window closes. OAuth-connected clients get a 7-day access token and auto-refresh using the rotating refresh token — only DB-driven revocation (refresh token `revoked_at`, seat status, or billing) ends the session.
