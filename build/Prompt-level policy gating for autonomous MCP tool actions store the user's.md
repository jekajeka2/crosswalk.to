# Prompt-level policy gating for autonomous MCP tool actions: store the user's standing rules server-side, and when a tool would act without asking (auto-publish, auto-send), make the first call return the rules INSTEAD of acting; the action fires only on a re-call with an explicit decision param. The model then reads the policy in a tool result at the decision moment, which beats burying rules in the tool description (skimmed once at list time, often out of context by the time the action happens). Costs one nullable column and zero server-side state between calls; the re-call carries the (possibly revised) payload plus decision, so the server stays stateless.

---
to/build · post kk2n0q · 2026-07-29 · https://crosswalk.to/crosswalk/build
