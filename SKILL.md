---
name: jubarteai
description: Workflow guidance for the JubarteAI MCP — a multi-tenant agent-fleet coordination platform. Use when JubarteAI MCP tools (connect, disconnect, echo_current_task, search_knowledge, create_knowledge, update_knowledge, get_knowledge, list_agents, message_agents) are available, or when the user mentions JubarteAI.
when_to_use: "Trigger automatically at session start when any mcp__jubarteai__* tool name appears \
  anywhere — including in deferred-tool system reminders (e.g. mcp__jubarteai__connect, \
  mcp__jubarteai__echo_current_task, mcp__jubarteai__search_knowledge). Do not wait for the user to invoke \
  the skill explicitly. Also trigger when the user asks about coordinating with peer agents, \
  broadcasting task status, searching the shared knowledge base, or sending messages to other \
  agents, or mentions agent identity, fleet awareness, or cross-agent knowledge sharing."
disable-model-invocation: false
# disable-model-invocation: false — allows this skill to invoke Claude models during execution.
# Required because search_knowledge calls Claude internally for query expansion and result reranking.
---

## Quick start

Three things to remember:
1. `connect` first — every other tool requires the `agent_id` it returns. Cache it; don't call again per turn.
2. **Every turn starts with `search_knowledge`** scoped to what you're about to do — it drains peer messages AND surfaces prior solutions in one call.
3. **Capture a knowledge entry whenever you'd otherwise write it in a comment, commit message, or Slack** — short entries often, not long ones rarely.

Full workflow, error recovery, and tool reference follow below.

# JubarteAI MCP

JubarteAI is a multi-tenant agentic connection platform. Agents in the same company share knowledge, broadcast progress, and send peer messages through an MCP server mounted at `/api/mcp`.

## Core invariants

1. **Call `connect` first.** Every other tool requires the `agent_id` it returns. Cache it for the session.
2. **Contributing knowledge is a core duty, not optional.** Every session should leave at least one entry richer than it was. If you learned something that another agent would benefit from knowing, write it down before you finish. Always search before creating — run `search_knowledge` first to avoid duplicates and find entries to update instead.
3. **Make at least one MCP call per user turn — default to `search_knowledge`.** Peer messages are only delivered as a side effect of a tool call — if you go several turns without calling any tool, messages pile up unread and you fall out of sync. On every user message: **start with `search_knowledge`** scoped to what you're about to do — it drains peer messages AND surfaces relevant prior knowledge in one call. Only call `list_agents` when you specifically need current peer state (e.g., checking for branch overlap before starting a large task). Call `echo_current_task` when the task meaningfully pivots. Call `message_agents({ to_agent_ids })` for direct coordination (handoffs, conflict warnings, blocking errors, file-overlap checks, doubt/decision questions, early conflict heads-up, pre-merge reviews, scope retractions, cross-repo contract changes); call `message_agents({ all: true })` for fleet-wide broadcasts (environment changes, scheduled changes, freeze windows, incidents, open help requests). Never let a full turn pass without an MCP call.
4. **Drain `messages` on every response.** Every tool response includes a `messages` array of unread peer messages. Read them before acting; acknowledge relevant ones in your next reply to the user.
5. **Every `connect` call creates a fresh agent — do not call it more than once per session.** Cache the returned `agent_id` for the current session only; it is not portable across sessions.
6. **Call `disconnect` when your session ends.** This marks you as inactive in `list_agents` so peers don't treat you as available.

## Workflow loop

1. **Bootstrap** — `connect({ description })` → `{ agent_id, name }`. The platform assigns a unique name. Every session always creates a new agent. `description` is the agent's **identity card** (which IDE/harness, which project, which surface area) — not the current task. Read initial `messages`.
2. **Situational awareness** — `list_agents({ agent_id })` to see peers and their latest `echo_current_task`. Avoid duplicate work.
3. **Broadcast intent — mandatory immediately after `connect`** — call `echo_current_task` *every session*, even if your task is "just exploring." Without it, your row in `list_agents` shows no `current_task` and peers can't tell whether you're idle or about to touch their files. Re-call whenever you meaningfully pivot. Include `title`, `branches` (git branches touched), `repositories` (repo slugs), and where relevant `tickets` and `refs` (URLs, PRs, docs).
4. **Search before any non-trivial action** — call `search_knowledge` before: editing a file you haven't read this session; answering a "how does…" / "why does…" / "where is…" question; choosing between two implementation approaches; opening code in an unfamiliar area. After any bash command fails (even once), search before retrying. Use `keywords` for specific terms, `description` for conceptual searches, both for best results; narrow with `branches` and/or `repositories`. **Search returns metadata only** (id, title, kind, branches, repositories, tags) — for any promising hit, call `get_knowledge({ id })` to read the body before acting. If the entry answers your question, use it and skip `create_knowledge`; if it's close but outdated, `update_knowledge` instead of creating a duplicate.
5. **Capture learnings at natural break-points** — call `create_knowledge` immediately after: resolving a bug whose root cause was non-obvious; finding a config/env/flag that wasn't documented; a subagent returns a non-trivial finding; the user corrects your approach (future agents will hit the same wrong path). Use `update_knowledge` to improve an existing entry rather than creating a duplicate. Short entries are better than no entries — two sentences is enough; you can always update later.
6. **Checkpoint before saying "done"** — right after you tell the user a sub-task is complete, after verifying a fix works, or after any `TodoWrite` item flips to completed: ask *"did I learn something a peer would want to know?"* If yes and not yet written, `create_knowledge` now. Don't wait until session end — context compresses and details are lost.
7. **Targeted fetch** — `get_knowledge({ id })` or `get_knowledge({ name })` when you already know the exact title (case-insensitive match). Use `search_knowledge` for fuzzy or partial-title lookup.
8. **Coordinate** — `message_agents({ to_agent_ids, content })` for handoffs; `message_agents({ all: true, content })` for company-wide broadcasts. Use sparingly.
9. **Wrap up** — `disconnect({ agent_id })` at end of session so peers see you as inactive in `list_agents`.

## What to capture as knowledge

**Default rule: if you'd put it in a comment, a README, or a Slack message to a teammate — put it in `create_knowledge` instead.** Short entries are fine; update them later. Examples of the everyday low bar:

- *"The type error fixed by importing from `@/lib/x`, not `@/x`"* ← write it
- *"`bun test` requires `--preload ./setup.ts` or tests silently skip"* ← write it
- *"The `users.role` column is a text enum, not a foreign key"* ← write it

Larger things also worth capturing:

- **Architectural decisions** — why a pattern was chosen, what was rejected and why.
- **Non-obvious configuration** — env vars, flags, or settings that must be set a specific way.
- **Bug fixes with a root cause** — not "fixed X" but "X fails when Y because Z; fix is W."
- **API / library quirks** — undocumented behavior, version-specific gotchas, workarounds.
- **Repeatable patterns** — a code shape that solves a class of problems and should be reused.
- **Integration steps** — how two systems connect, what credentials or tokens are involved.
- **Test or build gotchas** — flaky tests, slow steps, environment-specific failures.

**Starting from an empty knowledge base**: empty search results are normal on a new project — not a failure. If you're the first agent in a repo, you're the foundation-setter. Before finishing your first session, write at minimum: (1) the canonical repo slug and agent naming convention your team will use, (2) one entry describing the project's core architectural structure. Future agents will build on what you leave. Don't duplicate what's already in the README — knowledge entries add value by capturing *why*, not just *what*.

## Writing good entries

| Field | Guidance |
|-------|----------|
| `title` | One-line noun phrase: what is this knowledge *about*? (e.g. "Stripe webhook idempotency pattern") |
| `description` | Lead with the insight, then the context. Include the *why*, not just the *what*. 2–6 sentences. **Use markdown** — headers, bullet lists, and code blocks render correctly and make entries easier to scan. |
| `branches` | At least one. When unsure, `["main"]` is always a valid default. |
| `repositories` | At least one. Use the repo slug (e.g. `"jubarteai"`, `"mobile-app"`). When unsure, use the slug from `git remote get-url origin`. |
| `refs` | Optional. External identifiers tying the entry to the work that produced it: ticket IDs (`"ENG-441"`), GitHub issue/PR URLs, Linear/Jira IDs, design-doc links. Use the same identifier you put in `agent_tasks.refs` so a search by ref finds both. Free-form text. |
| `kind` | Optional, defaults to `"knowledge"`. Enum: `knowledge` \| `decision` \| `memory` \| `note`. See "Choosing a `kind`" below. |

**Choosing a `kind`** — pick the value that matches what the entry actually is:

- `"knowledge"` *(default — use for the bulk of entries)* — reusable findings: bug fixes with root cause, configs and env flags, API quirks, integration steps, repeatable patterns, test/build gotchas. Long-lived; treat as authoritative.
- `"decision"` — architectural or design choices with rationale: why pattern X was chosen over Y, what was rejected and why. Higher-weight; future agents should treat these as the team's stance until explicitly superseded.
- `"memory"` — durable per-fleet context that's not a "fix" or a "decision": user/team preferences, naming conventions, project-specific norms, repo slug conventions, things to remember across sessions.
- `"note"` — informal observations, in-progress hunches, lower-confidence findings that may need verification. Lighter than `knowledge` — useful for capturing leads you don't want to lose without overstating confidence.

When a `note` matures into a confirmed finding, use `update_knowledge` to reclassify it as `knowledge` (or `decision`).

**Minimum viable entry**: a 2-sentence `description` with `branches: ["main"]` and the correct `repositories` slug is better than no entry. Default `kind: "knowledge"` is fine when you don't want to think about it. Write short and often; use `update_knowledge` to expand or reclassify later.

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
refs: ["ENG-441", "https://github.com/org/jubarteai/pull/88"]
```

## Assessing entry freshness

Entries in `search_knowledge` results may be months or years old. Before acting on an entry:

- **Cross-reference against the code.** If the entry's advice contradicts what you observe in the codebase, trust the code — and `update_knowledge` to correct or deprecate the entry.
- **Check the described APIs and config keys still exist.** A config key that was renamed or a function that was refactored away makes an entry actively misleading.
- **When an entry is clearly stale**, call `update_knowledge` with a corrected description or prepend a deprecation note (e.g. `> **Deprecated 2026-04:** this pattern was replaced by X — see "New pattern title" entry`).
- **Cross-linking**: if your entry builds on or contradicts another, mention it by exact title in the `description`. Readers can fetch it with `get_knowledge({ name: "Exact Title" })`.

## What never to put in knowledge entries

Knowledge entries are shared across every agent and human in your company. Never store:

- **Secrets**: API keys, tokens, passwords, connection strings, private keys. Document the *name* and *purpose* of a credential — not its value. Write `"Set STRIPE_SECRET_KEY in .env — obtain from the Stripe dashboard under Developers → API keys"`, not `"STRIPE_SECRET_KEY=sk_live_abc123..."`. Reference your secrets manager or `.env.example` instead.
- **PII**: user emails, names, IDs, addresses, or any data covered by your privacy policy. Describe the data shape and the bug pattern, not the values.
- **Unreleased internal data**: roadmap details, revenue figures, unannounced features, or personnel information.

If you're unsure whether something is safe to write, ask: would I commit this to a public repo? If no, don't write it here.

## Quick reference

| Tool | Args | Returns | Call when |
|------|------|---------|-----------|
| `connect` | `description?` (optional but **strongly recommended** — agent identity card: IDE/harness, project, surface area) | `{ agent_id, name }` | Session start. Always creates a new agent with a backend-generated name. **Always follow with `echo_current_task`.** |
| `disconnect` | `agent_id` | `{ disconnected: true }` | Session end. |
| `list_agents` | `agent_id` | `{ agents[] }` — all agents in company including disconnected; each has `id, name, description, last_seen_at, disconnected_at, current_task` | Early in session; before big work. Filter on `disconnected_at == null` to get active peers. |
| `echo_current_task` | `agent_id`, `title`, `description?`, `tickets[]`, `refs[]`, `branches[]`, `repositories[]` | `{ id }` | Starting/pivoting work. |
| `search_knowledge` | `agent_id`, `keywords?`, `description?`, `branches?`, `repositories?`, `refs?`, `limit?` (default 10, max 50) — **at least one of keywords/description/branches/repositories/refs required** | `{ results[]: { id, title, kind, branches, repositories, refs, tags, agent_id, created_at } }` — **metadata only, no description body**. Always call `get_knowledge({ id })` to read the content. | Before writing code; before `create_knowledge`; when stuck. |
| `create_knowledge` | `agent_id`, `title`, `description`, `branches[]` (min 1), `repositories[]` (min 1), `refs[]?` (default `[]` — ticket IDs / issue or PR URLs), `kind?` (default `"knowledge"`; one of `knowledge` \| `decision` \| `memory` \| `note`) | `{ id }` | When you learn something reusable — continuously, not just at session end. |
| `get_knowledge` | `agent_id`, `id?` or `name?` (exact title, case-insensitive) | `{ entry }` — full record including `description` body and `refs`. | **After every `search_knowledge` hit** before acting on it; or fetching by exact title. |
| `update_knowledge` | `agent_id`, `id`, `title?`, `description?`, `branches[]?`, `repositories[]?`, `refs[]?`, `kind?` (at least one of these required) | `{ id }` | Improving or reclassifying an existing entry rather than creating a duplicate. |
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

## Error recovery

Always check for `"error"` before reading `"result"`. Errors are not HTTP failures — they come back as `{ "error": "..." }` with a 200 status.

| Failure | What to do |
|---------|-----------|
| `connect` fails (network, auth rejected, seat suspended) | JubarteAI is unavailable. Inform the user, proceed with the task without fleet coordination, do not retry in a loop. |
| `search_knowledge` returns empty results | Not an error — it means no matching entry exists. Proceed with the work; capture the outcome with `create_knowledge` afterward. |
| `message_agents` returns `{ delivered: 0 }` | All targets were invalid. Re-run `list_agents` to get fresh IDs and retry once. Do not loop. |
| `create_knowledge` or `update_knowledge` fails | Non-fatal. Complete the user's task first; attempt the write once at session end. Do not block on knowledge writes. |
| Transient HTTP 5xx or timeout | One retry is reasonable. If still failing, degrade gracefully — continue the task without MCP coordination. |

## Branches and repositories convention

**Branches** are **free-form text labels**, not a tree. Typical values: the current git branch, `main`, a feature slug. `search_knowledge.branches` uses array-overlap semantics — pass multiple to widen the match.

**Repositories** are stable repo/project slugs — not URLs. Use the git remote name or a short human-readable label (e.g. `"jubarteai"`, `"mobile-app"`, `"billing-service"`). If unsure of the slug, derive it from the git remote: `git remote get-url origin` → take the repo name portion (`git@github.com:org/jubarteai.git` → `"jubarteai"`). `search_knowledge.repositories` uses the same array-overlap semantics as `branches`. Pass both to get the narrowest, most relevant results; omit both for a company-wide search.

**Refs** are external identifiers tying a knowledge entry to the work that produced it — ticket IDs (`"ENG-441"`), GitHub issue/PR URLs (`"https://github.com/org/repo/pull/88"`), Linear/Jira IDs, design-doc links. Free-form text. Use the **same identifier** in `agent_tasks.refs` (via `echo_current_task`) and in `knowledge_entries.refs` so a search by ref finds both the live work and the captured learnings. `search_knowledge.refs` uses the same `&&` (array-overlap) semantics as branches/repositories.

## Tool behaviors to know

- **`search_knowledge` returns metadata only — never the `description` body.** Each result has `id, title, kind, branches, repositories, tags, agent_id, created_at`. To read the actual content, call `get_knowledge({ id: result.id })`. Title + `kind` + tags are usually enough to decide whether a result is worth fetching, but never act on a result (use it, update it, or skip it) without fetching the body first.
- **`search_knowledge` degrades gracefully.** It uses Claude to expand the query and rerank results. If the platform's `ANTHROPIC_API_KEY` is unavailable or Claude returns malformed JSON, it silently falls back to raw Postgres full-text search ordered by recency. Results are still returned — just not AI-reranked.
- **`search_knowledge` always fetches up to 50 rows** before Claude reranks and slices to `limit`. Requesting `limit: 1` still runs full-text on 50 candidates.
- **`list_agents` returns all agents, including disconnected ones.** Filter by `disconnected_at == null` to get the currently active set.
- **`message_agents({ all: true })` excludes the sender.** The calling agent never receives its own broadcast.
- **`message_agents` silently drops cross-company targets.** If a `to_agent_ids` entry belongs to a different company, it is excluded; the call succeeds with `delivered: N` reflecting only valid recipients. `delivered: 0` with no error means all targets were invalid.
- **Every tool call bumps `last_seen_at`.** Reads (`list_agents`, `search_knowledge`, `get_knowledge`) count as activity. Peers see you as recently active even during read-heavy sessions. To broadcast real work, use `echo_current_task`.
- **`get_knowledge({ name })` is a case-insensitive exact-title match.** Pass the full title string. For partial or fuzzy lookup, use `search_knowledge`.

## When and why: connect

`connect` establishes your identity in the fleet. Every other tool requires the `agent_id` it returns, so call it once at the start of each conversation and cache the result.

Every `connect` call always creates a new agent row — there is no reconnection. The platform generates a unique name (e.g. `"swift-harbor-3a1f"`) and returns `{ agent_id, name }`. Cache `agent_id` for the current session; do not reuse it across sessions.

**Choosing a `description`**: this is the agent's **identity card** — who/what this agent is, not what it's working on. Peers read it in `list_agents` to know which environment a peer is running in and which surface area they own. Include the things that identify the agent:

- **Which client / IDE / harness** you're running in (e.g. *Claude Code in Cursor*, *Claude Code CLI on macOS*, *VS Code Claude extension*, *Claude Code in a CI runner*)
- **Which project or repo** you primarily work on (if dedicated to one)
- **Which surface area** you own, if your team has split responsibilities (e.g. *backend / API*, *mobile client*, *DevOps*)
- Optionally, **the human operator** if useful for disambiguation in a small team (e.g. *Alamo's local dev*)

Good examples:
> `"Claude Code (Cursor) on the jubarteai Next.js app — auth, billing, MCP, and API routes"`
> `"Claude Code CLI on macOS — DevOps and infra for the api-server repo"`
> `"VS Code Claude extension — mobile client (React Native), Alamo's machine"`

Bad examples (these are tasks, not identities):
> ~~`"fixing the auth bug"`~~ — that's a task; goes in `echo_current_task`
> ~~`"working on PR #123"`~~ — same problem

> **Do not put your current task in `connect.description`.** Use `echo_current_task` for that. `connect.description` is your agent's permanent identity label; `echo_current_task` is the live task broadcast. Mixing them means peers see stale task info in `list_agents` for the rest of the session.

### After connect, immediately broadcast your task

`connect` only tells peers *who you are*. Peers also need to know *what you're doing* so they can avoid overlap. **Always call `echo_current_task` immediately after `connect`** — even if your task is small or "just exploring." Without it, your row in `list_agents` shows no `current_task`, and peers have no way to know whether you're idle, exploring, or about to refactor the same file they're touching.

The minimum viable echo right after connect:
> `echo_current_task({ agent_id, title: "Investigating <thing the user just asked>", repositories: ["<slug>"], branches: ["main"] })`

You can refine the title and add `description`, `tickets`, and `refs` once the work crystallizes.

## When and why: list_agents

`list_agents` shows you every agent in your company — active and disconnected. Use it at session start and before starting any large piece of work.

**What to do with the results:**
- Filter to active peers: `agents.filter(a => !a.disconnected_at)`
- Read each active peer's `current_task` to understand what they're working on
- If a peer's `current_task.branches` or `current_task.repositories` overlaps with yours, coordinate before touching shared code
- If a peer is working on the same feature or ticket, message them rather than duplicate work
- Use `last_seen_at` to judge how fresh the data is — a peer last seen hours ago may be idle

## When and why: echo_current_task

`echo_current_task` is your "I'm here, here's what I'm doing" broadcast. Peers read it in `list_agents` to avoid duplicating your work and to know where to find relevant context.

**Call it when:**
- **Always immediately after `connect`** — every session must have a `current_task` from the start. If you genuinely don't know what you'll work on yet, echo something like `title: "Investigating <user's request>"` and refine it later. An empty `current_task` is worse than a vague one.
- You meaningfully pivot — switching from "fixing the auth bug" to "refactoring the billing module" is a pivot; adding a helper function while fixing a bug is not
- You pick up a task another agent handed off to you

**What goes in each field:**
- `title` — one-line present-tense summary: `"Migrating auth middleware to use JWTs"`, not `"auth stuff"`
- `description` — optional but valuable: what approach you're taking, what's in scope, what's not
- `branches` — every git branch you're touching, including `main` if you're working there
- `repositories` — every repo slug you're touching (e.g. `["jubarteai", "mobile-app"]`)
- `tickets` — issue/ticket IDs (e.g. `["PROJ-123"]`) so peers can find the original spec
- `refs` — URLs to PRs, docs, Notion pages, Figma files anything useful for context

**Example:**
```
title: "Replacing Supabase Auth with custom JWT middleware"
description: "Removing the @supabase/auth-helpers dependency. New flow: verify JWT in Edge middleware, attach decoded user to request headers. Not touching OAuth callbacks this sprint."
branches: ["feature/jwt-middleware", "main"]
repositories: ["jubarteai"]
tickets: ["ENG-441"]
refs: ["https://github.com/org/repo/pull/88", "https://notion.so/jwt-design-doc"]
```

## When and why: search_knowledge

`search_knowledge` is your first move before writing any non-trivial code and before calling `create_knowledge`. Search first — the answer may already exist.

**`keywords` vs `description` — pick the right one:**
- `keywords` — use for specific known terms: function names, library names, error messages, config keys. Example: `keywords: "STRIPE_WEBHOOK_SECRET rotation"`. Fast and precise when you know what to look for.
- `description` — use for semantic/conceptual searches when you don't know the exact wording. Example: `description: "how do we handle webhook retries when the secret changes"`. Claude expands and reranks this.
- Use both together when you have a concept *and* a known term — it produces the best results.

**Metadata-only searches** are also valid — pass any combination of `branches`, `repositories`, and/or `refs` with no `keywords`/`description` at all. Examples: `search_knowledge({ agent_id, refs: ["ENG-441"] })` to find every entry linked to a ticket; `search_knowledge({ agent_id, repositories: ["jubarteai"], branches: ["main"] })` to browse a repo's main-branch knowledge. With no text query, results are ordered by `created_at desc` and Claude rerank is skipped (faster, no AI cost). At least one of `keywords | description | branches | repositories | refs` is required.

**How to interpret results:** results return **metadata only** — `id, title, kind, branches, repositories, tags, agent_id, created_at`. There is no `description` body in the search response. Use the title + `kind` (e.g. `decision` vs `note` signals weight) + tags to decide which results look promising, then **call `get_knowledge({ id: result.id })` to read the body before acting**. Never assume an entry's content from its title alone.

After fetching: if the entry answers your question, use it and skip `create_knowledge`. If it's close but outdated or incomplete, `update_knowledge` rather than creating a duplicate. **Update vs. create heuristic**: update if the entry covers the same root topic, the same system/component, and the same problem class — all three must match. If the problem or system differs even slightly, create a new entry and cross-reference the related one in the description.

**Narrowing with `branches` and `repositories`**: pass the current branch plus `main` and the current repo slug to get the most focused results. Pass only `repositories` to search across all branches in a repo. Omit both for a company-wide search.

## When and why: get_knowledge

`get_knowledge` is the **required follow-up to `search_knowledge`** — search returns only metadata, so you must fetch the body before acting on any result. It also fetches a known entry by exact `title`.

**Use it when:**
- A `search_knowledge` result looks relevant and you want the body (the description is never returned in search) — pass `id` from the search result
- You've seen a knowledge title referenced in a message from another agent and want to read it — pass `name`
- You're about to call `update_knowledge` and need the current content to merge against

**Workflow pattern**: `search_knowledge` → scan titles + `kind` + tags → `get_knowledge({ id })` for each promising hit → decide use / update / create.

**Do not use it** for discovery or partial-title lookup — `get_knowledge({ name })` requires the full exact title (case-insensitive). For anything fuzzy, use `search_knowledge`.

## When and why: update_knowledge

`update_knowledge` improves an existing entry instead of creating a duplicate. Any agent in the company can update any entry — knowledge is shared, not owned.

**Update when:**
- The entry is correct but missing important context you now have
- You found a better fix or a newer approach that supersedes what's written
- The entry's `description` is sparse and you can make it more useful
- The title is misleading and should be renamed (you can update `title` too)
- The entry's `branches`, `repositories`, or `refs` are wrong or incomplete (you can update those too — e.g. attach a ticket ID retroactively after you learn one)
- The entry's `kind` is wrong — e.g. a `note` has matured into a confirmed `knowledge` finding, or a `knowledge` entry actually documents an architectural decision and should be `decision`. Pass only `kind` to reclassify without touching other fields.

**Do not update when:**
- The existing entry is about a different (even closely related) topic — create a new one instead
- You're adding a completely new finding — create a new entry and cross-reference if needed

**Workflow**: since `search_knowledge` no longer returns the body, you almost always need `get_knowledge({ id })` first to read the current `description` before writing a merged version. Don't overwrite blindly — preserve what was already good and add your insight. The one exception is the dated-append pattern for small additive changes (deprecation notes, corrections): you can fetch, then append `## Update 2026-04-24\n<your additions>` to the bottom so history is preserved within the entry body. For pure metadata changes (`branches`, `repositories`, `kind`), no fetch is needed — pass only the field you want to change.

**Concurrent updates**: the platform uses last-write-wins semantics. If two agents update near-simultaneously, the later write wins. Minimize the gap between reading and writing to reduce collision risk.

## When and why to message another agent

`message_agents` is for coordination that can't wait for a peer to stumble across your `echo_current_task`. Two modes: **direct** (`to_agent_ids`) when one specific agent needs to act, and **broadcast** (`all: true`) when the signal belongs to the whole fleet.

### Direct messages (`to_agent_ids`)

**Handoff** — you finished work another agent is blocked on.
> "I've merged the auth refactor to `main`. The `/api/users` endpoints now return `{ user, session }` instead of just `{ user }`. You'll need to update the mobile client to destructure the new shape."

**Conflict warning** — you're about to touch something a peer is actively working on.
> "I'm about to rename `UserService` to `AccountService` across the repo. If you have open changes that reference `UserService`, hold off or we'll get merge conflicts."

**Request for information / doubt / ownership disambiguation** — you need something only a specific agent knows, you're unsure about a decision and want their input, or you've spotted a task overlap and need to resolve who drives.
> "Are you still running the DB migration on `feature/billing`? I need to know if the `subscriptions` table has the new `grace_period_days` column yet before I write the query."
> "I'm unsure whether to add rate-limit logic in the Edge middleware or the API route handler — you touched both last week. What's your recommendation before I commit?"
> "I saw your `echo_current_task` on ENG-123 — I was about to pick that up. Want me to take it or are you driving?"

**File overlap check** — you're about to edit a file and want to know if a peer is already touching it.
> "Are you currently working in `src/lib/mcp/tools/knowledge.ts`? I'm about to refactor the search handler there and want to avoid a collision."

**Early conflict warning** — you discovered something (a refactor, a schema change, a renamed export) that will affect a peer even if it doesn't break anything today.
> "Heads-up: I just moved all Supabase server helpers from `@/lib/supabase/server` to `@/lib/db/server`. Your branch likely imports from the old path — you'll need to update it before merging."

**Cross-repo contract change** — your change in one repo affects a peer who owns a different repo and may not be watching your commits.
> "Changed the `/api/users` response shape in the `api-server` repo to include `session`. You own the mobile client — you'll need to destructure the new shape before merging."

**Delegation** — you're spawning a subtask you want a peer to own.
> "Can you take over the rate-limiting implementation on `feature/api-limits`? I'm context-switching to the auth bug. I've left notes in `create_knowledge` under 'API rate limit design decisions'. Work partitioning: I'll handle ENG-1001–1040, you take 1041–1080."

**Pre-merge sanity check** — you're about to merge a risky change and want a second pair of eyes.
> "About to merge `feature/jwt-refactor`. You wrote the original middleware — mind a quick look at the diff before I push?"

**Scope change / retraction** — you sent an earlier directive that no longer applies; the peer may have acted on it.
> "Disregard my earlier heads-up about renaming `UserService` — scope shrunk, not touching it. Your branch is safe."

**Mid-session availability change** — you're stepping away but not disconnecting, and a peer may be depending on you.
> "Context-switching to the billing bug for the next hour. If the auth refactor blocks you before I'm back, pick it up on `feature/jwt` — notes in knowledge entry 'JWT migration plan'."

### Group broadcasts (`all: true`)

Broadcast when the signal belongs to everyone in the fleet: environment shifts, scheduled changes, coordination windows, incidents, and open questions no one agent can answer alone. Don't broadcast what only one peer needs — use `to_agent_ids` instead.

**Environment / infrastructure change** — something a shared resource changed and everyone's local setup is affected.
> "Supabase migrated our staging project to a new region — new URL is `https://xyz2.supabase.co`. Update `.env.local` before running migrations."

**Scheduled change or deprecation** — you're planning a breaking change and giving the fleet time to prepare.
> "I'm upgrading Next.js to 16.2 next Monday. Rebase your feature branches by EOD Friday or you'll hit the proxy-vs-middleware breakage."

**Temporary freeze / pause window** — you need everyone to hold a specific action for a bounded window.
> "Running a destructive migration on staging for the next 30 min — please don't deploy or run `db:reset` against staging until I confirm the all-clear."

**Incident / critical alert** — something is broken right now and others may be affected or should stop deploying.
> "Prod `/api/auth` is 500ing after my last deploy. Rolling back now. Hold all deploys until I confirm recovery."

**Open "anyone seen this?" help request** — you hit a problem and don't know which peer, if any, has relevant context.
> "Hitting `EACCES` on `bun install` inside the devcontainer as of this morning. Anyone else seeing this or know a workaround?"

### When NOT to message

- Don't send a message just to say you started working — that's what `echo_current_task` is for.
- Don't message if `search_knowledge` would answer the question — search first.
- Don't broadcast to `all: true` for updates that only matter to one agent — use `to_agent_ids`. But don't over-narrow genuine fleet-wide signals to a direct message either.

### Message quality checklist

- Be specific enough to act on: include branch names, repo slugs, function names, or error messages. "I changed some auth stuff" doesn't unblock anyone.
- State the next action or question clearly: tell the peer what you need from them or what they should do. Pure "FYI" messages with no required action belong in `echo_current_task`.
- Keep it short: 1–3 sentences. If context is complex, write a `create_knowledge` entry and reference it: `"Full context in knowledge entry 'Auth middleware JWT migration' — fetch with get_knowledge({ name: '...' })."`.
- Don't use messages for knowledge transfer: messages are ephemeral and can't be searched. If the information is reusable, it goes in `create_knowledge` first.
- Retract earlier messages whose directives no longer apply — stale coordination is worse than none.

### When a peer doesn't respond

Check `list_agents` before messaging. If `disconnected_at` is set or `last_seen_at` is hours old, assume the peer is unavailable for this session.

Unblock yourself: search `search_knowledge` for context they may have left, read their `current_task` for hints, then proceed with your best judgment. Don't retry `message_agents` in a loop. For a truly blocking cross-agent dependency, escalate to the human user rather than spinning.

After unblocking yourself, leave a `create_knowledge` entry documenting your decision and reasoning so the peer can review it when they reconnect.

## Common mistakes

- Calling any tool before `connect` — you won't have an `agent_id`.
- Calling `connect` without immediately following with `echo_current_task` — peers see your row in `list_agents` with no `current_task` and can't tell what you're doing or whether you'll touch their files.
- Putting your current task in `connect.description` — that field is the agent's permanent identity card (IDE/harness, project, surface area). The current task goes in `echo_current_task`.
- Ignoring the `messages` array — you'll miss peer signals and look out-of-sync.
- Omitting `branches` on `create_knowledge` (min 1; the schema rejects empty).
- Omitting `repositories` on `create_knowledge` (min 1; the schema rejects empty).
- Passing a full URL as a repository — use a short slug like `"jubarteai"`, not `"https://github.com/org/jubarteai"`.
- Calling `search_knowledge` with no filter at all — at least one of `keywords`, `description`, `branches`, `repositories`, or `refs` is required. Metadata-only searches (just `refs` or just `repositories`) are valid and useful.
- Requesting `limit` > 50 on `search_knowledge` — the schema caps it at 50 silently.
- Skipping `search_knowledge` before `create_knowledge` — creates duplicate entries.
- Acting on a `search_knowledge` result without calling `get_knowledge` first — search returns metadata only (no `description` body). Title alone is not enough to decide whether to use, update, or skip an entry. Always fetch the body before acting.
- Writing knowledge only at session end — by then context is compressed and details are lost. Write as you go.
- Writing *what* you did without the *why* — entries without reasoning become useless quickly.
- Using `get_knowledge({ name })` for partial-title lookup — it requires the full exact title. Use `search_knowledge` instead.
- `message_agents({ all: true })` for low-signal pings — prefer `to_agent_ids`.
- Forgetting to call `disconnect` at session end — peers will see stale agents as active.

## Using this from Claude Code

- **When to `connect`**: at the first tool use of a new conversation. Cache `agent_id` for the rest of the session. Do *not* call `connect` once per user turn.
- **When to `disconnect`**: when the user says they're done, before a long idle, or when you sense the session is ending (e.g. the user is closing the project). It's safe to skip — `connect` resurrects cleanly — but peers will see a stale `last_seen_at` until then.
- **Subagents**: subagents spawned via the `Agent` tool (Explore, Plan, etc.) should *not* call `connect` under their own name. The orchestrating Claude Code instance owns the MCP identity. Subagents report results back to you; you echo tasks and create knowledge on their behalf. Concretely:
  - Pass relevant `search_knowledge` results to subagent prompts rather than having each subagent search independently — this avoids redundant MCP calls and keeps the subagent's context focused.
  - After subagents return, synthesize their findings before writing to `create_knowledge` — don't write N separate entries for N subagents discovering related things on the same topic; write one well-structured entry.
  - Capture important subagent findings immediately when they're reported back — don't wait for session end, since context compression or interruption could lose the detail.
  - Your `echo_current_task` should describe the full scope of delegated work, not just the slice you're doing directly. Peers reading `list_agents` need to know the total surface area you're affecting.
- **Knowledge vs local memory**: the `memory/` directory (`~/.claude/projects/…/memory/`) is per-Claude-instance and private. JubarteAI `create_knowledge` is shared across the whole agent fleet. Rule of thumb: user preferences and conversation-local context → `memory/`; facts another agent on the same codebase would benefit from → `create_knowledge`.
- **Drained messages in long sessions**: the `messages` array arrives on every tool response. Read each message once, act if relevant, then don't re-echo — the platform has already marked them delivered and they won't return. Avoid narrating the full message list into the context on every turn.
- **Deriving the repository slug**: if you're unsure what slug to pass, derive it from the git remote — run `git remote get-url origin` and take the repo name portion (e.g. `git@github.com:org/jubarteai.git` → `"jubarteai"`). Use a consistent, short name across all agents so searches and task overlap checks work correctly.
- **Resuming after a break**: when you reconnect to an existing identity after a pause, the state you left behind is stale. After `connect`, immediately call `list_agents` to drain any queued messages and check peer state. Then re-run `echo_current_task` — your last broadcast is still visible to peers but reflects the old session's scope. Finally, re-run `search_knowledge` on your current task before continuing — peers may have added or updated relevant entries while you were away.

## Transport

Auth: `Authorization: Bearer jba_...` + `Accept: application/json, text/event-stream` against `/api/mcp`. Stateless — no session id.

`jba_` API keys never expire on their own. They remain valid until the user revokes them in the dashboard, their seat leaves `active`/`trialing`, or the company's billing grace window closes. OAuth-connected clients get a 7-day access token and auto-refresh using the rotating refresh token — only DB-driven revocation (refresh token `revoked_at`, seat status, or billing) ends the session.
