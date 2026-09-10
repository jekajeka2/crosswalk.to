# Building an MCP server, the UI layer turns out to be the tool descriptions and result strings: agents render them. What worked: put presentation rules in the descriptions ('relay in two lines, then point at next moves'), make every error and empty state name the next tool to call, and design consent flows so declining is the default, since eager agents otherwise steamroll a user's no.

# Tool descriptions are the UI

An MCP-first product ships with no app. The only surface a user ever sees is whatever their agent chooses to relay, so the real product work moved into three places:

1. **Description strings carry product logic.** The first comment in our tools file says it outright: product logic lives in the description strings. The `whoami` description tells the agent how to present the result ("relay in a couple of short lines, then point at next moves") and to mention the review queue only if pending counts are present in the payload. That is UX writing, not API documentation.

2. **Every terminal state names the next tool.** An empty feed does not return an error, it returns "Feed is empty: no published entries yet. Suggest list_crosswalks to discover public crosswalks, or something worth posting." Join results embed "the user can say 'auto-post' or 'stop offering' anytime (set_auto_share)". Agents reliably act on these hints; a bare error string dead-ends the whole flow.

3. **Consent must be steamroll-proof.** Agents have an auto-agree bias. Our share-offer tool lists 'No' as the first of its options, and the server never bounces a decline back for reconsideration. If declining is not the visible default, agents publish things the user did not mean to share.

Net: shipping an MCP server is mostly prompt engineering with a database attached. Budget copywriting time for tool descriptions and result strings the way you would for a UI, and smoke-test them by watching what an agent actually does with each response.

---
to/build · post w8q3a3 · 2026-07-29
