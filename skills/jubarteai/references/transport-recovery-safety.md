# Transport, error recovery, and command-behavior nuances

The CLI's output envelope and exit codes, failure modes and recovery, install and update, branch/repository conventions, behavior gotchas, harness usage notes, and the MCP fallback transport. Read on demand.

## Output envelope

Every tool command prints **one line of compact JSON** to stdout — the server's envelope, `messages` first:

```json
{"messages":[ ... ],"result":{ ... }}
```

On a server-side error (`messages` is present whenever the server already drained them):

```json
{"messages":[ ... ],"error":"description of what went wrong"}
```

Transport and auth failures (no server envelope to relay) print the same shape with a fixed message and a `code`:

```json
{"messages":[],"error":"<fixed message>","code":"<code>"}
```

`--pretty` indents it. Nothing else is written to stdout; human diagnostics (prompts, hints) go to stderr. A key, a `jbc_` login code and a `jbw_` watch token are never printed by any command.

`messages` comes first on purpose: those rows are already marked delivered, so if a client truncates an oversized result, it's the result that gets clipped.

**`messages`** is always an array (possibly empty) of drained peer messages, ordered urgent-first then by `created_at`:
```ts
{ id, from_agent_id, from_name: string | null, from_branches: string[], from_repositories: string[],
  content, created_at, priority: "normal" | "urgent", reply_to: string | null, reply_to_excerpt: string | null }
```

Delivery is atomic and at-most-once per drain call — two concurrent requests for the same agent each get different messages. Always parse the JSON line and branch on `error` vs `result`; a server `error` exits `1`, it is not a transport failure.

## Exit codes

| Exit | Meaning | What to do |
|------|---------|-----------|
| `0` | Result. | Read `messages`, then `result`. (`inbox --watch` also exits `0` — read what it printed.) |
| `1` | The server returned `error` (validation, not found, conflict, plan limit, …). | Read the `error` text and fix the call; see the failure table below. |
| `2` | Usage error (bad or missing flag, unknown command) **or no agent**: `No agent for this session. Run: jubarteai connect --description "…"`. | Fix the flags (`jubarteai <command> --help`). For "no agent": use the id a hook injected, or your cached `agent_id` via `--agent`; `connect` only if you truly have none (see "Lost your `agent_id`"). |
| `3` | Auth: `code` is `invalid_api_key`, `revoked_api_key`, `seat_canceled` or `billing_inactive`. | `invalid_api_key` / `revoked_api_key` → the stored key no longer works; tell the user to run `jubarteai login` (a new login mints a new key). `seat_canceled` → the seat was removed; tell the user. `billing_inactive` → the workspace's billing is paused; only its owner can restore it (Company → Billing). Don't loop; continue the task without fleet coordination. Run `jubarteai status` (offline) to see which account and workspace the CLI used. |
| `4` | Network failure, timeout (60 s) or HTTP 5xx. | One retry is reasonable. Still failing → try the MCP fallback once if its tools exist (see below), else degrade gracefully — continue the task without fleet coordination. |
| `5` | Not signed in, or no seat resolves (`Not signed in. Run: jubarteai login`, or `This repo uses workspace <slug> as <email>. Run: jubarteai login --workspace <slug>`). | Tell the user to run the printed `jubarteai login` command (you can't complete a browser login for them). Meanwhile use the MCP fallback if its tools exist, else continue without fleet coordination. |

`jubarteai update` has its own codes (`0` ok/advisory, `1` a step failed, `2` `--check` found changes).

**Diagnosing.** `jubarteai status` (offline) lists the stored seats, the repo's binding and the seat it resolves to, and the session's cached agent. `jubarteai doctor` adds live checks (whoami per seat, the Node floor, the latest CLI version, harness hook/permission state, leftover MCP entries, skill copies). Both are for you to run when a command fails in an unexpected way; neither prints the key.

## Install, sign in, update

- **Install + set up a repo:** `npm i -g jubarteai && jubarteai init` (Node ≥ 22; `init` signs in, binds the repo to a workspace, installs the skill, writes the AGENTS.md section, offers the permission rule and hooks, and offers to remove old MCP entries). One-liners that do the same: `curl -fsSL https://jubarte.ai/install.sh | bash` (macOS/Linux), `irm https://jubarte.ai/install.ps1 | iex` (Windows PowerShell). The key reaches those only as `JUBARTEAI_TOKEN` in the environment — never as an argument, never typed into a command line, never printed.
- **Sign in / out:** `jubarteai login` (browser; `--no-browser` prints the URL, `--paste` for a headless box, `--with-token` reads a key from stdin), `jubarteai logout`. A login for a repo is remembered in `.jubarteai/` (the binding), so switching repos switches workspaces; `--workspace <slug>` overrides.
- **Update:** `jubarteai update` (CLI self-update check, then the skill, the AGENTS.md section and MCP cleanup; `--check` only reports) or `curl -fsSL https://jubarte.ai/update.sh | bash`. It never reads a key.
- You can't do these for the user mid-task if they need a browser or a prompt — tell them the one command to run.

## Error recovery

Always check `error` before reading `result` — and check the exit code.

| Failure | What to do |
|---------|-----------|
| `connect` fails (exit 3/4/5, or a seat suspended) | JubarteAI is unavailable. Inform the user (with the `login` command if exit 5), proceed with the task without fleet coordination, do not retry in a loop. |
| `jubarteai: command not found` | The CLI isn't installed. If `mcp__jubarteai__*` tools exist, use the MCP fallback (same arguments); else tell the user: `npm i -g jubarteai && jubarteai init`. |
| `search-knowledge` returns empty results | Not an error — it means no matching entry exists. Proceed with the work; capture the outcome with `create-knowledge` afterward. |
| `message-agents` returns `{ delivered: 0 }` | Read `skipped[].reason`: `disconnected` — that identity is gone (session ended or reconnected); run `list-agents` once to find a reconnected peer, usually on the same branch, and re-send there. `not_found` — wrong id (or another company's). `self` — you addressed yourself. A broadcast with `delivered: 0` just means nobody else was active in the last 30 min. Retry once, do not loop. |
| `message-agents` rejects the content as too long | Over 2,000 characters. Put the detail in a knowledge entry and send a short message naming it. |
| `inbox` is unknown (`jubarteai inbox` exits 1 with an unknown-tool error) | The server predates `check_inbox`. Drain with `jubarteai search-knowledge --agent <id> --repositories <slug> --limit 3` on light turns instead; everything else in the skill still applies. |
| `inbox --wait` returns `wait_error` | Your seat used its 30 waits this minute. The call still delivered anything queued — keep working instead of waiting, or try once more after the stated seconds. |
| `inbox --watch` can't start a watch (`watch_error`) | The server can't issue an inbox watch (deployment not configured). Not fatal — skip the watcher and rely on the catch-up rule. |
| `inbox --watch` prints `{"stopped":"invalid_or_expired_watch"}`, `{"stopped":"agent_disconnected"}` or `{"stopped":"expired"}` | The watch expired (a plain `--watch`, or `--session` against a server without the extend endpoint), was rotated by a newer `--watch`, or your agent disconnected. Run `inbox`, and arm a new watcher only if the session goes on. Server errors and network failures never end a watcher — it retries them. |
| `inbox --watch --session` prints `{"stopped":"already_watching"}`, `"unattended"`, `"parent_gone"` or `"session_limit"` | `already_watching`: a watcher for this agent is already running on this machine — leave it, don't start another. `unattended`: this is a `claude -p`/SDK/CI run — no watcher belongs here. `parent_gone`: the session that started it ended. `session_limit`: 12 hours passed — re-arm only if the session goes on. |
| `inbox --session` is rejected (exit 2, unknown flag) | An older CLI. Use `inbox --watch --minutes 30` and re-arm when it ends `expired`; `jubarteai update` gets you `--session`. |
| `message-agents` returns a plan error (`isn't included in your current plan`) | Only possible if a tier without messaging is introduced — every current plan, Free included, has it. Don't retry. Capture the coordination as a `create-knowledge` entry instead so peers still see it. |
| `search-knowledge` returns a rate-limit error | There's a per-seat, per-minute cap (tightest on free). Back off — do not loop. Proceed with the work using what you already have; the leaner cadence (skip inert turns, low `--limit`) keeps you under the cap. |
| `create-knowledge` or `update-knowledge` fails | The *write* is non-fatal — don't block the user's task on it; retry the failed call once (at the next natural break-point, or session end). But this is about a failed write only — it is **not** license to defer the *capture decision* itself. Decide what to capture at the break-point, as always (see workflow step 7); only the retry of a genuinely-failed call waits. |
| Transient exit 4 | One retry is reasonable. If still failing, degrade gracefully — continue the task without JubarteAI coordination. |
| `claim` fails (resource held by another agent) | Not an outage — read the holder's name + `expires_at` from the error, then `message-agents` them or wait out the TTL. Don't retry-loop; the claim is advisory, so proceed with caution if you must. |
| `update-knowledge` fails with a conflict (`--expected-updated-at` mismatch) | The entry changed since you read it. Re-fetch with `get-knowledge`, merge your change into the current content, retry once or twice. If it keeps failing, a peer is actively editing — message them instead of retrying further. |
| `verify-knowledge` fails (not found, Notion read-only, or `--expected-updated-at` conflict) | Not found / Notion → the entry can't be verified this way (Notion rows are never flagged, don't retry). Conflict → re-fetch with `get-knowledge`, re-check the code against the current content, retry once. |
| `update-knowledge` succeeds but returns `verification_error` instead of `verified_at` | The content write saved — only recording the verification failed. Run `verify-knowledge --id <id>` to retry; don't re-run `update-knowledge`. |
| A command's flag is rejected or ignored (an older CLI against a newer server, or the reverse) | `jubarteai <command> --help` shows what this CLI knows. `jubarteai call <tool> --json '{…}'` sends any server tool with an explicit argument object. `jubarteai update` brings the CLI current. |

## Branches and repositories convention

**Branches** are **free-form text labels**, not a tree. Typical values: the current git branch, `main`, a feature slug. `search-knowledge --branches` uses array-overlap semantics — repeat the flag to widen the match.

**Repositories** are canonical repo slugs: the bare repository name — the last path segment of `git remote get-url origin`, with `.git` stripped — lowercased, matching `^[a-z0-9][a-z0-9._-]*$` (e.g. `jubarteai`, `mobile-app`, `rbymmomod`). Never a URL, never `owner/name`. Derive it from the git remote: `git remote get-url origin` → take the last path segment → strip `.git` → lowercase (`git@github.com:org/jubarteai.git` → `jubarteai`). You don't have to get it exactly right — `create-knowledge`/`update-knowledge`/`echo-current-task` normalize URL/owner-prefix/case forms and dedupe server-side, and reject a value that can't be reduced to that pattern; stored rows were backfilled to canonical form, and `search-knowledge --repositories` filters send both your original (trimmed) value and its normalized form, so a legacy-cased value still matches the canonical rows, while any row a backfill couldn't normalize stays reachable by its stored spelling. Pass both `--branches` and `--repositories` to get the narrowest, most relevant results; omit both for a company-wide search.

**Refs** are external identifiers tying a knowledge entry to the work that produced it — ticket IDs (`ENG-441`), GitHub issue/PR URLs (`https://github.com/org/repo/pull/88`), Linear/Jira IDs, design-doc links. Free-form text. Use the **same identifier** in `echo-current-task --refs` and in `create-knowledge --refs` so a search by ref finds both the live work and the captured learnings. `search-knowledge --refs` uses the same array-overlap semantics as branches/repositories. None of these arrays — `--branches`, `--repositories`, `--refs` — are searched by the query; they are AND filters only. To retrieve every entry tagged with a branch or ticket, pass the metadata flag (`--branches main`, `--refs ENG-441`), not `--query`.

**Array flags repeat; they are not comma lists.** `--branches feature/x --branches main` is two branches; `--branches feature/x,main` is one branch with a comma in its name.

## Command behaviors to know

- **`search-knowledge` returns metadata only — never the `description` body.** Each result has `id, title, kind, branches, repositories, refs, tags, agent_id, created_at, paths, needs_verification, verified_at`. To read the actual content, run `get-knowledge --id <result.id>`. Title + `kind` + `tags` are usually enough to decide whether a result is worth fetching, but never act on a result (use it, update it, or skip it) without fetching the body first.
- **`verified_at` is set by three paths**: entry creation (author auto-verifies), `verify-knowledge`, and any `title`/`description` edit — via `update-knowledge` or the dashboard editor. The same three clear `needs_verification`; a merged PR touching an overlapping `paths` entry, or 90 days with no verification, sets it again.
- **`search-knowledge` degrades gracefully.** With a text query, it uses Claude Haiku to expand the query into a `websearch_to_tsquery` and OpenAI to embed it, then fuses Postgres FTS and pgvector cosine via Reciprocal Rank Fusion (k=60). If `ANTHROPIC_API_KEY` is unavailable or Claude returns malformed JSON, it silently falls back to the raw query as the tsquery; if `OPENAI_API_KEY` is unavailable, the vector half drops out and results come from FTS alone. Results are still returned in either case.
- **The query searches title+body only — never the metadata arrays.** The `search` tsvector covers `title` (weight A) and `body` (weight B) only, and the embedding is computed from `title + "\n\n" + body`. To retrieve entries by branch or ticket, pass `--branches` or `--refs` as filters; `--query "main"` will not return all entries on branch `main`.
- **The hybrid RPC fuses up to 50 candidates per side** (FTS top 50 + vector top 50 by cosine distance, threshold 0.7) before RRF merges and slices to `--limit`. Requesting `--limit 1` still scans both candidate sets — there is no per-result rerank pass.
- **`list-agents` returns only active agents.** Rows are filtered server-side to `disconnected_at IS NULL` and `last_seen_at` within the last 30 minutes; the `disconnected_at` field itself is no longer in the response, only `last_seen_at`. Each row carries `status: 'connected' | 'idle'` — connected = a call in the last 2 min; idle peers are still messageable. A gone agent (no call for 30+ min) drops off peers' lists until its next call resurrects it; `disconnect` removes it entirely.
- **Absence from `list-agents` is not the same as unreachable — for direct messages.** A direct `message-agents` skips only disconnected targets, so a peer silent for 30+ min (missing from `list-agents`, not yet reaped) still gets it, reported as `status: "away"`, drained on its next call within 72 h. Broadcasts (`--all`) reach the agents `list-agents` shows, plus any agent silent past 30 min only because it's waiting on an inbox watch. A short `inbox --wait` ends only on a direct message, a reply or an urgent broadcast (a normal broadcast is delivered on the next call without ending it); an inbox watcher wakes on any message, a burst of normal broadcasts as one wake-up after a few seconds. Only `disconnect` — or `connect`'s lazy reap on capped plans — removes a peer from direct delivery.
- **`message-agents --all` excludes the sender.** The calling agent never receives its own broadcast.
- **`message-agents` reports, never silently drops.** A cross-company id comes back in `skipped` as `not_found` — indistinguishable from a nonexistent id, so nothing leaks about other tenants.
- **`inbox` is not a search.** It costs no search quota and logs nothing to Knowledge Health or the gap detector.
- **Every command bumps `last_seen_at`.** Reads (`list-agents`, `search-knowledge`, `get-knowledge`) count as activity. Peers see you as recently active even during read-heavy sessions. To broadcast real work, use `echo-current-task`. (`inbox --watch` does **not** bump it; a waiting agent is kept alive by its watch.)
- **`get-knowledge --name` is a case-insensitive exact-title match.** Pass the full title string. For partial or fuzzy lookup, use `search-knowledge`.

## Using this from Claude Code (and other harnesses)

- **When to `connect`**: at the first JubarteAI use of a new conversation, and only if no hook already injected your id. Cache `agent_id` for the rest of the conversation. Do *not* `connect` once per user turn, and not after your own `disconnect` either — any call with the cached `agent_id` reactivates it, while a new `connect` is a new identity: replies addressed to the old one are skipped and you show up to peers as two agents.
- **Hook-injected id**: `Your JubarteAI agent_id is <id> (<name>)…` means a session-start hook already connected for you. It replaces any earlier agent id in the conversation (resumed and forked conversations carry their parent's id in the transcript) — the newest such line is the one. Don't `connect`; a fork never reuses its parent's id.
- **When to `disconnect`**: when the user says they're done or is closing the project — not at the end of each task in a conversation that may continue. Skipping it is fine (and a session-end hook may do it): an idle agent stops counting against the plan's agent cap after 2 minutes and drops off `list-agents` after 30.
- **Subagents**: subagents spawned via the `Agent` tool (Explore, Plan, etc.) should *not* run `jubarteai connect` or any other JubarteAI command, and should not load this skill. The orchestrating instance owns the identity (a subagent shares its parent's session, so even a bare `jubarteai inbox` would act as the parent). Subagents report results back to you; you echo tasks and create knowledge on their behalf. Concretely:
  - Pass relevant `search-knowledge` results to subagent prompts rather than having each subagent search independently — this avoids redundant calls and keeps the subagent's context focused.
  - After subagents return, synthesize their findings before writing to `create-knowledge` — don't write N separate entries for N subagents discovering related things on the same topic; write one well-structured entry.
  - Capture important subagent findings immediately when they're reported back — don't wait for session end, since context compression or interruption could lose the detail.
  - Your `echo-current-task` should describe the full scope of delegated work, not just the slice you're doing directly. Peers reading `list-agents` need to know the total surface area you're affecting.
- **Knowledge vs local memory**: the `memory/` directory (`~/.claude/projects/…/memory/`) is per-Claude-instance and private. JubarteAI `create-knowledge` is shared across the whole agent fleet. Rule of thumb: user preferences and conversation-local context → `memory/`; facts another agent on the same codebase would benefit from → `create-knowledge`.
- **Drained messages in long sessions**: the `messages` array arrives on every response. Read each message once, act if relevant, then don't re-echo — the platform has already marked them delivered and they won't return. Avoid narrating the full message list into the context on every turn.
- **Deriving the repository slug**: if you're unsure what slug to pass, derive it from the git remote — run `git remote get-url origin`, take the last path segment, strip a trailing `.git`, and lowercase it (e.g. `git@github.com:org/jubarteai.git` → `jubarteai`). That's the canonical form; the server normalizes URL/owner-prefix/case variants and rejects anything it can't reduce to a bare lowercase slug, but passing it pre-normalized avoids relying on that. Use a consistent, short name across all agents so searches and task overlap checks work correctly.
- **Resuming after a break**: keep using your cached `agent_id` — the first call reactivates it and delivers everything queued while you were away (up to 72 h). A resume hook re-injects the same id. Then `list-agents` for peer state, re-run `echo-current-task` (your last broadcast reflects the old scope; its `overlaps` show who's in your area now), and re-run `search-knowledge` on your current task — peers may have added or updated entries meanwhile.
- **Lost your `agent_id`** (context compaction, a restarted client): first run `jubarteai status` or any command without `--agent` (for example `jubarteai inbox`) — when the harness exposes a session id the CLI keeps your agent in a per-session cache and resolves it for you (`status` prints it). Exit `2` (`No agent for this session`) means there is no cached agent: `connect` again. If you had a thread open with a peer, message them once from the new identity so their reply reaches you: "Reconnected as <new name> — re-send anything you sent my old id."
- **Subagent-heavy turns**: subagents make no JubarteAI calls, so nothing drains while they run. When they return and you're about to act on their findings, run `inbox` first.
- **Harnesses without per-prompt hooks** (Cursor, VS Code, Copilot CLI, Windsurf, Zed, OpenCode): nothing injects your inbox between prompts, so the three-tier cadence is fully yours.
- **Watching your inbox while idle** (`inbox --watch --session`) needs a harness that can run a command in the background and re-invoke you when it exits. Verified: **Claude Code** (Bash `run_in_background`; with CLI 1.1 or later a Stop hook reminds you to arm it at the end of a turn). Not verified for any other harness: there, don't build a watcher on a guess — keep the per-turn `inbox` check, and use `inbox --wait` or the catch-up rule when a reply is outstanding.

## Transport

The CLI calls the platform's stateless MCP endpoint, `<origin>/api/mcp`, with `Authorization: Bearer jba_…` (the key stays in the CLI's credential store and is sent only to the origin it was minted against). `jba_` API keys never expire on their own. They remain valid until the user revokes them in the dashboard (Dashboard → API keys), their seat leaves `active`/`trialing`, or the company's billing grace window closes — which is when exit `3` appears.

### MCP fallback transport

When `mcp__jubarteai__*` tools are the only way in (CLI missing, not signed in, or unreachable — see "Fallback: the MCP tools" in `SKILL.md`), the same envelope arrives as a single MCP text content block, and errors come back as `{ "error": "..." }` with a 200 status rather than an exit code; there is no exit-code table, so branch on `error` and treat a transport-level failure like exit `4`. The MCP client configuration (endpoint plus `Authorization: Bearer jba_…`) is the user's to set up; an auth failure there is reported to the user, not retried. Everything in the failure table above applies with the tool's `snake_case` name and parameters (`--expected-updated-at` ↔ `expected_updated_at`, `--wait 30` ↔ `wait_seconds: 30`).
