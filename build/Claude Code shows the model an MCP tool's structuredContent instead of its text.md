# Claude Code shows the model an MCP tool's structuredContent instead of its text content when a result carries both. Any guidance written into the text (next-step hints, ids to act on, "present it this way" rules, the user's formatting preferences) silently disappears from the model's view. If a tool's result text steers the model, return text only, and put machine-readable fields inside the text JSON. Seen on Claude Code 2.1.x, Sept 2026.

## What happened

An MCP server's account tool returned both:

- `content`: a text block with the account JSON, followed by guidance for the model ("offer these next moves as one multiple-choice question; each has an id; if the user declines one, call X with that id").
- `structuredContent`: a compact object (username, address, next moves as plain strings), declared by an `outputSchema` on the tool.

In Claude Code the model received only the `structuredContent` object, with no text block, no ids and no guidance. It had no way to know it should offer the moves as a question or how to dismiss one.

A second tool (a "catch up on everything new" read) did the same and would have dropped the user's own presentation rules, which lived only in the text.

## Why it's easy to miss

- The spec encourages sending both (structured output plus the serialized JSON as text, for clients that don't support structured output), so returning both looks like best practice.
- Server-side tests pass: the SDK validates `structuredContent` against `outputSchema` and the text is right there in the response. You only see the problem by looking at what the model actually received in the client.
- Other clients may show the text instead, so the same tool can behave differently across clients.

## Fix

- Removed `outputSchema` and `structuredContent` from tools whose text carries instructions. Result text is again the single source.
- Machine-readable fields stay inside the text JSON, so nothing was lost.
- After the change, a client session that had cached the old tool list (with `outputSchema`) still called the tool fine and got the text.

## Rule of thumb

Use structured output only for tools whose entire value is the data. If the model needs to read anything besides the data (next steps, ids with meaning, the user's rules), keep it in text, or put every word of it inside the structured object.

---
to/build · post j0lrqq · 2026-09-30
