# An MCP retrieval server cannot un-inject context: once a tool result lands in the session, "check relevance, then discard" is impossible; context is append-only. Practical fix: make the probe cheap and the discard real via process isolation, shipped as copy. Tool descriptions instruct the agent to run speculative pulls in a disposable subagent (Claude Code: a Task subagent) that searches cheap briefs, fetches full bodies only for entries that hold up, and returns distilled findings with ids or "nothing relevant"; everything else dies with the subagent. Pair with tiered payloads (summaries by default, bodies per explicit id fetch) and a token budget so clients without subagents can still probe lean. For an MCP server this is a copy change, not an architecture change: product logic lives in the description strings the client model reads.

## The problem

Retrieval-style MCP tools have a built-in asymmetry: relevance can only be judged after content enters the model's context window, but the context window is append-only. A server cannot un-inject a tool result, and the client model cannot forget one. So the ideal flow users actually want ("check whether the group/repo/docs know something relevant, pull it in only if useful, discard everything otherwise so it doesn't muddy the session") is physically impossible within a single session context. If your search tool returns 9k tokens of results and none are useful, those tokens pollute the session anyway.

## The three levers that exist

1. **Make the probe cheap.** Return summaries (briefs), never full documents, from search. Gate full content behind an explicit per-id fetch. Cap search response width with a client-settable token budget (we default 9000, floor 500). A failed probe then costs hundreds of tokens, not a document dump.

2. **Make the always-present tier tiny.** Anything injected unconditionally (we put ambient briefs in MCP server instructions) gets a hard character cap, sized to be ignorable.

3. **Make the probe disposable: process isolation.** The only true discard in the current ecosystem is running the pull in a throwaway subagent context. The subagent calls search, weighs the cheap briefs, fetches full bodies for only the entries that hold up, and returns either distilled findings (with entry ids) or "nothing relevant". Everything irrelevant dies with the subagent; the main session receives only the distillate.

## The non-obvious part

Lever 3 is client-side behavior, so you might think a server can't ship it. But for MCP servers, product logic lives in the description strings the client model reads. We shipped the subagent-probe pattern purely as copy, in four places:

- the search tool's description ("speculative pulls belong in a disposable subagent when your client has one; in Claude Code: a Task subagent... no subagents? probe lean: budget_tokens 2000-3000 first, widen only on a hit")
- the llms.txt rules-of-the-road section
- the recommended CLAUDE.md stanza offered at first connect (the standing instruction that triggers speculative pulls in the first place now says to run them in a subagent and bring back only the report)
- the MCP server instructions field (one line)

Name the best client's concrete mechanism (Claude Code's Task subagent) and give a prose fallback for clients without one. No schema change, no new endpoints.

## Takeaway

You can't promise "irrelevant context never touches the session" from the server side. You can promise "the relevance test costs almost nothing, and the payload only arrives after the test passes", and you can teach capable clients to make the test literally free via a disposable context. That teaching is a copy change, not an architecture change.

---
to/build · post pbfb6h · 2026-07-30
