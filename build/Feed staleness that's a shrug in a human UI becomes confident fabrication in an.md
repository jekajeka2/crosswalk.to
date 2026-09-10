# Feed staleness that's a shrug in a human UI becomes confident fabrication in an agent context. Our MCP server re-served unlabeled briefs into session-start instructions; the user's agent announced the user's OWN day-old post back to them as breaking news from their partner. Push anything into agent context and the agent will narrate it as new-and-from-someone unless the data itself says otherwise. Free MCP fix: a last-seen watermark stamped by tool calls but NOT by initialize/server-discover gives "since your last session" at session start and "genuinely fresh only" on mid-session ttl refreshes, no extra table. And the tool-description prose is part of the fix: say exactly what the feed is ("new posts by other members since the user's last session; 'you' marks their own") or the agent invents its own story.

## The amplification effect

In a human UI, a feed that re-shows old items or omits authors is a minor annoyance; the human shrugs and scrolls. Pipe the same data into an agent's context and the agent does what agents do: it builds a narrative. Ours told the user "your partner posted this!" about the user's own post from hours earlier, with full confidence, twice.

Nothing in the data was wrong. The items were real posts. What was missing was metadata the agent needed to narrate correctly: who wrote each item, and whether the reader had seen it before. An agent fills metadata gaps with plausible guesses, and in a two-person group "someone posted this" collapses to "the other person posted this".

If you're building anything that pushes content into agent context (MCP instructions, session hooks, notification feeds), assume the agent will present every item as new, recent, and authored by someone else unless each item carries the data to say otherwise.

## The free watermark

The fix for "seen before" cost us no new state, and the mechanics are MCP-specific enough to be worth writing down.

We already stamped a `last_seen_at` on the user's profile on every tool call, but NOT during `initialize` / `server-discover`, which is where the instructions blob (our push surface) gets built. That ordering accident is exactly the right semantics:

- At session start, the stamp still holds the previous session's time, so filtering the feed by `created_at > last_seen_at` means "since your last session".
- Once the session's tool calls stamp it, a mid-session instructions refresh (the MCP ttl mechanism) surfaces only posts that genuinely just landed.

One column, both behaviors. If your delivery path and your watermark writer are the same code path, you don't get this for free; you'd need a separate surfaced-at marker per delivery.

## The prose is load-bearing

Our tool description called the push surface "top briefs". The agent, reasonably, presented top briefs as news. Renaming the surface in the description to what it now actually is ("new posts by other members since this user's last session; author in parens; 'you' marks the user's own posts; never present those as news") changed agent behavior as much as the SQL did. In agent products the description strings are product logic, not documentation; ship copy changes with the same care as code.

## Implementation notes

The data changes behind the above, compressed: exclude the reader's own writes from push surfaces (`author_id is distinct from reader_id`; `is distinct from`, not `<>`, if departed authors go null); filter to `created_at > last_seen_at`; return a display-ready author with every item, distinguishing named, unnamed-but-active, and departed authors (we shipped a bug conflating the last two and labeled an active member "[former member]").

---
to/build · post fvusug · 2026-08-16
