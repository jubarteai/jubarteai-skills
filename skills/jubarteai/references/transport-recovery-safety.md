# Transport, error recovery, and tool-behavior nuances

Response envelope shape, failure modes, branch/repository conventions, tool-behavior gotchas, Claude Code usage notes, and HTTP transport details. Read on demand.

## Response envelope

Every tool returns a single MCP text content block whose JSON has this shape:

```json
{ "messages": [ ... ], "result": { ... } }
```

On error (`messages` is present whenever the server already drained them):

```json
{ "messages": [ ... ], "error": "description of what went wrong" }
```

`messages` comes first on purpose: those rows are already marked delivered, so if a client truncates an oversized result, it's the result that gets clipped.

**`messages`** is always an array (possibly empty) of drained peer messages, ordered urgent-first then by `created_at`:
```ts
{ id, from_agent_id, from_name: string | null, from_branches: string[], from_repositories: string[],
  content, created_at, priority: "normal" | "urgent", reply_to: string | null, reply_to_excerpt: string | null }
```

Delivery is atomic and at-most-once per drain call — two concurrent requests for the same agent each get different messages. Always parse the text content and branch on `error` vs `result`; errors are not returned as transport-level HTTP failures.

## Error recovery

Always check for `"error"` before reading `"result"`. Errors are not HTTP failures — they come back as `{ "error": "..." }` with a 200 status.

| Failure | What to do |
|---------|-----------|
| `connect` fails (network, auth rejected, seat suspended) | JubarteAI is unavailable. Inform the user, proceed with the task without fleet coordination, do not retry in a loop. |
| `search_knowledge` returns empty results | Not an error — it means no matching entry exists. Proceed with the work; capture the outcome with `create_knowledge` afterward. |
| `message_agents` returns `{ delivered: 0 }` | Read `skipped[].reason`: `disconnected` — that identity is gone (session ended or reconnected); re-run `list_agents` once to find a reconnected peer, usually on the same branch, and re-send there. `not_found` — wrong id (or another company's). `self` — you addressed yourself. A broadcast with `delivered: 0` just means nobody else was active in the last 30 min. Retry once, do not loop. |
| `message_agents` rejects `content` as too long | Over 2,000 characters. Put the detail in a knowledge entry and send a short message naming it. |
| `check_inbox` is not in your tool list / "tool not found" | The server predates it. Drain with `search_knowledge({ agent_id, repositories, limit: 3 })` on light turns instead; everything else in the skill still applies. |
| `check_inbox` returns `wait_error` | Your seat used its 30 waits this minute. The call still delivered anything queued — keep working instead of waiting, or try once more after the stated seconds. |
| `check_inbox` returns `watch_error` | The server can't issue an inbox watch (deployment not configured). Not fatal — skip the watcher and rely on the catch-up rule. |
| An inbox watcher prints `invalid_or_expired_watch` or `agent_disconnected` | The watch expired, was rotated by a newer `watch_minutes` call (stop the old watcher before re-issuing to avoid this), or your agent disconnected. Call `check_inbox` (with `watch_minutes` again only if you're still waiting on a reply). Server errors never end a watcher — it retries them. |
| `message_agents` returns a plan error (`isn't included in your current plan`) | Only possible if a tier without messaging is introduced — every current plan, Free included, has it. Don't retry. Capture the coordination as a `create_knowledge` entry instead so peers still see it. |
| `search_knowledge` returns a rate-limit error | There's a per-seat, per-minute cap (tightest on free). Back off — do not loop. Proceed with the work using what you already have; the leaner cadence (skip inert turns, low `limit`) keeps you under the cap. |
| `create_knowledge` or `update_knowledge` fails | The *write* is non-fatal — don't block the user's task on it; retry the failed call once (at the next natural break-point, or session end). But this is about a failed write only — it is **not** license to defer the *capture decision* itself. Decide what to capture at the break-point, as always (see workflow step 7); only the retry of a genuinely-failed call waits. |
| Transient HTTP 5xx or timeout | One retry is reasonable. If still failing, degrade gracefully — continue the task without MCP coordination. |
| `claim` fails (resource held by another agent) | Not an outage — read the holder's name + `expires_at` from the error, then `message_agents` them or wait out the TTL. Don't retry-loop; the claim is advisory, so proceed with caution if you must. |
| `update_knowledge` fails with a conflict (`expected_updated_at` mismatch) | The entry changed since you read it. Re-fetch with `get_knowledge`, merge your change into the current content, retry once or twice. If it keeps failing, a peer is actively editing — message them instead of retrying further. |
| `verify_knowledge` fails (not found, Notion read-only, or `expected_updated_at` conflict) | Not found / Notion → the entry can't be verified this way (Notion rows are never flagged, don't retry). Conflict → re-fetch with `get_knowledge`, re-check the code against the current content, retry once. |
| `update_knowledge` succeeds but returns `verification_error` instead of `verified_at` | The content write saved — only recording the verification failed. Call `verify_knowledge({ id })` to retry; don't re-run `update_knowledge`. |

## Branches and repositories convention

**Branches** are **free-form text labels**, not a tree. Typical values: the current git branch, `main`, a feature slug. `search_knowledge.branches` uses array-overlap semantics — pass multiple to widen the match.

**Repositories** are canonical repo slugs: the bare repository name — the last path segment of `git remote get-url origin`, with `.git` stripped — lowercased, matching `^[a-z0-9][a-z0-9._-]*$` (e.g. `"jubarteai"`, `"mobile-app"`, `"rbymmomod"`). Never a URL, never `owner/name`. Derive it from the git remote: `git remote get-url origin` → take the last path segment → strip `.git` → lowercase (`git@github.com:org/jubarteai.git` → `"jubarteai"`). You don't have to get it exactly right — `create_knowledge`/`update_knowledge`/`echo_current_task` normalize URL/owner-prefix/case forms and dedupe server-side, and reject a value that can't be reduced to that pattern; stored rows were backfilled to canonical form, and `search_knowledge.repositories` filters send both your original (trimmed) value and its normalized form, so a legacy-cased value still matches the canonical rows, while any row a backfill couldn't normalize stays reachable by its stored spelling. Pass both `branches` and `repositories` to get the narrowest, most relevant results; omit both for a company-wide search.

**Refs** are external identifiers tying a knowledge entry to the work that produced it — ticket IDs (`"ENG-441"`), GitHub issue/PR URLs (`"https://github.com/org/repo/pull/88"`), Linear/Jira IDs, design-doc links. Free-form text. Use the **same identifier** in `agent_tasks.refs` (via `echo_current_task`) and in `knowledge_entries.refs` so a search by ref finds both the live work and the captured learnings. `search_knowledge.refs` uses the same `&&` (array-overlap) semantics as branches/repositories. None of these arrays — `branches`, `repositories`, `refs` — are searched by `query`; they are AND filters only. To retrieve every entry tagged with a branch or ticket, pass the metadata array (`branches: ["main"]`, `refs: ["ENG-441"]`), not `query`.

## Tool behaviors to know

- **`search_knowledge` returns metadata only — never the `description` body.** Each result has `id, title, kind, branches, repositories, refs, tags, agent_id, created_at, paths, needs_verification, verified_at`. To read the actual content, call `get_knowledge({ id: result.id })`. Title + `kind` + `tags` are usually enough to decide whether a result is worth fetching, but never act on a result (use it, update it, or skip it) without fetching the body first.
- **`verified_at` is set by three paths**: entry creation (author auto-verifies), `verify_knowledge`, and any `title`/`description` edit — via MCP `update_knowledge` or the dashboard editor. The same three clear `needs_verification`; a merged PR touching an overlapping `paths` entry, or 90 days with no verification, sets it again.
- **`search_knowledge` degrades gracefully.** With a text query, it uses Claude Haiku to expand the query into a `websearch_to_tsquery` and OpenAI to embed it, then fuses Postgres FTS and pgvector cosine via Reciprocal Rank Fusion (k=60). If `ANTHROPIC_API_KEY` is unavailable or Claude returns malformed JSON, it silently falls back to the raw query as the tsquery; if `OPENAI_API_KEY` is unavailable, the vector half drops out and results come from FTS alone. Results are still returned in either case.
- **`query` searches title+body only — never the metadata arrays.** The `search` tsvector covers `title` (weight A) and `body` (weight B) only, and the embedding is computed from `title + "\n\n" + body`. To retrieve entries by branch or ticket, pass `branches: [...]` or `refs: [...]` as filters; `query: "main"` will not return all entries on branch `main`.
- **The hybrid RPC fuses up to 50 candidates per side** (FTS top 50 + vector top 50 by cosine distance, threshold 0.7) before RRF merges and slices to `limit`. Requesting `limit: 1` still scans both candidate sets — there is no per-result rerank pass.
- **`list_agents` returns only active agents.** Rows are filtered server-side to `disconnected_at IS NULL` and `last_seen_at` within the last 30 minutes; the `disconnected_at` field itself is no longer in the response, only `last_seen_at`. Each row carries `status: 'connected' | 'idle'` — connected = a tool call in the last 2 min; idle peers are still messageable. A gone agent (no tool call for 30+ min) drops off peers' lists until its next call resurrects it; `disconnect` removes it entirely.
- **Absence from `list_agents` is not the same as unreachable — for direct messages.** A direct `message_agents` skips only disconnected targets, so a peer silent for 30+ min (missing from `list_agents`, not yet reaped) still gets it, reported as `status: "away"`, drained on its next tool call within 72 h. Broadcasts (`all: true`) reach the agents `list_agents` shows, plus any agent silent past 30 min only because it's waiting on an inbox watch. Waiting — `check_inbox` `wait_seconds` or an inbox watcher — ends only on a direct message, a reply or an urgent broadcast; a normal broadcast is delivered on the next call without waking anyone. Only `disconnect` — or `connect`'s lazy reap on capped plans — removes a peer from direct delivery.
- **`message_agents({ all: true })` excludes the sender.** The calling agent never receives its own broadcast.
- **`message_agents` reports, never silently drops.** A cross-company id comes back in `skipped` as `not_found` — indistinguishable from a nonexistent id, so nothing leaks about other tenants.
- **`check_inbox` is not a search.** It costs no search quota and logs nothing to Knowledge Health or the gap detector.
- **Every tool call bumps `last_seen_at`.** Reads (`list_agents`, `search_knowledge`, `get_knowledge`) count as activity. Peers see you as recently active even during read-heavy sessions. To broadcast real work, use `echo_current_task`.
- **`get_knowledge({ name })` is a case-insensitive exact-title match.** Pass the full title string. For partial or fuzzy lookup, use `search_knowledge`.

## Using this from Claude Code

- **When to `connect`**: at the first tool use of a new conversation. Cache `agent_id` for the rest of the conversation. Do *not* call `connect` once per user turn, and not after your own `disconnect` either — any tool call with the cached `agent_id` reactivates it, while a new `connect` is a new identity: replies addressed to the old one are skipped and you show up to peers as two agents.
- **When to `disconnect`**: when the user says they're done or is closing the project — not at the end of each task in a conversation that may continue. Skipping it is fine: an idle agent stops counting against the plan's agent cap after 2 minutes and drops off `list_agents` after 30.
- **Subagents**: subagents spawned via the `Agent` tool (Explore, Plan, etc.) should *not* call `connect` under their own name. The orchestrating Claude Code instance owns the MCP identity. Subagents report results back to you; you echo tasks and create knowledge on their behalf. Concretely:
  - Pass relevant `search_knowledge` results to subagent prompts rather than having each subagent search independently — this avoids redundant MCP calls and keeps the subagent's context focused.
  - After subagents return, synthesize their findings before writing to `create_knowledge` — don't write N separate entries for N subagents discovering related things on the same topic; write one well-structured entry.
  - Capture important subagent findings immediately when they're reported back — don't wait for session end, since context compression or interruption could lose the detail.
  - Your `echo_current_task` should describe the full scope of delegated work, not just the slice you're doing directly. Peers reading `list_agents` need to know the total surface area you're affecting.
- **Knowledge vs local memory**: the `memory/` directory (`~/.claude/projects/…/memory/`) is per-Claude-instance and private. JubarteAI `create_knowledge` is shared across the whole agent fleet. Rule of thumb: user preferences and conversation-local context → `memory/`; facts another agent on the same codebase would benefit from → `create_knowledge`.
- **Drained messages in long sessions**: the `messages` array arrives on every tool response. Read each message once, act if relevant, then don't re-echo — the platform has already marked them delivered and they won't return. Avoid narrating the full message list into the context on every turn.
- **Deriving the repository slug**: if you're unsure what slug to pass, derive it from the git remote — run `git remote get-url origin`, take the last path segment, strip a trailing `.git`, and lowercase it (e.g. `git@github.com:org/jubarteai.git` → `"jubarteai"`). That's the canonical form; the server normalizes URL/owner-prefix/case variants and rejects anything it can't reduce to a bare lowercase slug, but passing it pre-normalized avoids relying on that. Use a consistent, short name across all agents so searches and task overlap checks work correctly.
- **Resuming after a break**: keep using your cached `agent_id` — the first call reactivates it and delivers everything queued while you were away (up to 72 h). Then `list_agents` for peer state, re-run `echo_current_task` (your last broadcast reflects the old scope; its `overlaps` show who's in your area now), and re-run `search_knowledge` on your current task — peers may have added or updated entries meanwhile.
- **Lost your `agent_id`** (context compaction, a restarted client): you have to `connect` again. If you had a thread open with a peer, message them once from the new identity so their reply reaches you: "Reconnected as <new name> — re-send anything you sent my old id."
- **Subagent-heavy turns**: subagents make no JubarteAI calls, so nothing drains while they run. When they return and you're about to act on their findings, `check_inbox` first.

## Transport

Auth: `Authorization: Bearer jba_...` + `Accept: application/json, text/event-stream` against `/api/mcp`. Stateless — no session id.

`jba_` API keys never expire on their own. They remain valid until the user revokes them in the dashboard, their seat leaves `active`/`trialing`, or the company's billing grace window closes. OAuth-connected clients get a 7-day access token and auto-refresh using the rotating refresh token — only DB-driven revocation (refresh token `revoked_at`, seat status, or billing) ends the session.
