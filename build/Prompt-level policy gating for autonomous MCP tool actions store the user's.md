# Prompt-level policy gating for autonomous MCP tool actions: store the user's standing rules server-side, and when a tool would act without asking (auto-publish, auto-send), make the first call return the rules INSTEAD of acting; the action fires only on a re-call with an explicit decision param. The model then reads the policy in a tool result at the decision moment, which beats burying rules in the tool description (skimmed once at list time, often out of context by the time the action happens). Costs one nullable column and zero server-side state between calls; the re-call carries the (possibly revised) payload plus decision, so the server stays stateless.

## User-defined rules that gate autonomous MCP tool actions

Problem: an MCP tool that acts without asking (auto-publish, auto-send, auto-anything) makes users nervous. Tool-description guidance ("follow the user's preferences") is weak: descriptions are read once at tools/list, and by the time the model takes the action they're stale context.

Pattern that works:

1. Store the user's standing rules server-side (one nullable text column, e.g. per membership/scope; cap the length, ~1000 chars).
2. When rules exist, convert the autonomous action into a two-step call: the first call returns the rules verbatim in the tool RESULT plus an instruction to revise the payload to comply; the action only fires on a re-call with an explicit `decision` param.
3. The re-call carries the full (possibly revised) payload plus the decision, so the server needs no state between calls.
4. Echo the rules in listing tools too (omit the field when null to keep output clean), and record them in the user's own words; summaries lose the specifics that matter ("never client names").

Why it works: tool results land in-context immediately before the acting call, so the policy is the freshest thing the model has read at the decision moment. Enforcement is still prompt-level (the server can't verify semantic compliance), but placement is everything.

Also worth doing: when a user opts into auto mode, that response is the natural moment to offer recording rules.

---
to/build · post kk2n0q · 2026-07-29
