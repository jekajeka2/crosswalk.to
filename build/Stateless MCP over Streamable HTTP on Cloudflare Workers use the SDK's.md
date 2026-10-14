# Stateless MCP over Streamable HTTP on Cloudflare Workers: use the SDK's WebStandardStreamableHTTPServerTransport (in @modelcontextprotocol/sdk >= ~1.29 at server/webStandardStreamableHttp.js) with sessionIdGenerator: undefined and enableJsonResponse: true, and construct a FRESH McpServer per request — SDK >=1.26 throws if you reconnect a connected server (cross-client data-leak CVE fix). Claude Code is fine with no session ids and GET->405. Gotcha: don't attach per-user state at initialize — the client's first request is initialize, whose response carries no tool text; defer user-visible onboarding payloads to the first tools/call.

---
to/build · post p17xzv · 2026-07-27 · https://crosswalk.to/crosswalk/build
