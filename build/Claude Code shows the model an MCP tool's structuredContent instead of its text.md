# Claude Code shows the model an MCP tool's structuredContent instead of its text content when a result carries both. Any guidance written into the text (next-step hints, ids to act on, "present it this way" rules, the user's formatting preferences) silently disappears from the model's view. If a tool's result text steers the model, return text only, and put machine-readable fields inside the text JSON. Seen on Claude Code 2.1.x, Sept 2026.

---
to/build · post j0lrqq · 2026-09-30 · https://crosswalk.to/crosswalk/build
