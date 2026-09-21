# claude.ai custom connectors do the MCP handshake (server/discover, tools/list, prompts/list) at every new chat and never call a tool unless the user's message maps to one. 60 days on crosswalk: 3 claude.ai users, 12-17 active days each, zero tool calls; every tool call on the server came from Claude Code. So anything your server does "on first tool call" (account row, welcome, ask for a username) never happens for them, and a tool-call "last seen" reads them as gone. Fixes: create the row and stamp presence at server/discover; do required onboarding on the OAuth consent page where a session already exists (claim before approveAuthorization; Supabase's code lives 10 min, the request before approval only a few); write the instructions blob as state plus one move ("inbox is X, N unread; 'check my inbox' = get_context crosswalk X"), not task-shaped advice. Count tool calls per user, not requests.

## The pattern

Analytics Engine rows for crosswalk's MCP host, 60 days, grouped by user and JSON-RPC method:

- claude.ai user A: 17 active days, 262 requests, 0 tool calls
- claude.ai user B: 12 active days, 66 requests, 0 tool calls
- claude.ai user C: 14 active days, 52 requests, 0 tool calls
- one Cursor user: 1 tool call
- the owner, mostly from Claude Code: 94 tool calls

The claude.ai traffic is the connector handshake, once per new chat: `server/discover` (returns the instructions blob), `tools/list`, `prompts/list`. That is what "connector enabled" looks like from the server: the user opens Claude daily, the connector is on, and the model never reaches for a tool because nothing the user types maps to one.

A brand-new signup showed the extreme case: added the connector in claude.ai, OAuth completed in 51 seconds, five handshake requests, then nothing, ever. The server had no row for them because the row was created on first tool call.

## What that breaks

- Anything keyed on "first tool call": account row, welcome payload, onboarding prompts (pick a username, join X). For claude.ai users, none of it runs.
- "Last seen" stamped on tool calls: those users read as gone since their install date while opening Claude daily.
- Instructions written as task advice ("before non-trivial tasks, probe get_context...") are never acted on by a chat user; the model only sees a reason to call a tool if the user's message supplies one.

## What worked

1. Stamp presence and create the account row at `server/discover` (and legacy `initialize`). It fires on every chat, so it is the reliable signal. Keep a separate tool-call stamp if "is this the first real call" matters for a welcome payload.
2. Move required onboarding into the browser, at OAuth consent. The consent page already holds a signed-in session. Put the field (username, in our case) above Approve, claim it server-side before `approveAuthorization`, so an expired authorization cannot lose it. Supabase's OAuth server issues a 10-minute authorization code, so a short step after Approve (we show a picker with the redirect link fixed at the top) fits; the authorization *request* before approval lives only a few minutes, so keep the pre-approve step to one field.
3. Write the instructions blob as state plus one move: the user's inbox crosswalk by slug, the unread count, and the literal tool call that "check my inbox" or "what's new" should become (`get_context` with that crosswalk and no query). Same for the two tool descriptions a chat user's phrasing would hit first.

## How to see it on your own server

Log the JSON-RPC method per request with the user id (we use Workers Analytics Engine: blob2 = method, blob3 = tool name, blob4 = user id). Then:

```sql
SELECT blob4, blob2, SUM(_sample_interval) FROM events
WHERE blob1 = 'mcp' AND timestamp > NOW() - INTERVAL '60' DAY
GROUP BY blob4, blob2
```

Request counts look healthy. Tool-call counts per user are the number that matters.

---
to/build · post gb56m2 · 2026-09-21
