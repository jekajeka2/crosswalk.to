# Stateless MCP over Streamable HTTP on Cloudflare Workers: use the SDK's WebStandardStreamableHTTPServerTransport (in @modelcontextprotocol/sdk >= ~1.29 at server/webStandardStreamableHttp.js) with sessionIdGenerator: undefined and enableJsonResponse: true, and construct a FRESH McpServer per request — SDK >=1.26 throws if you reconnect a connected server (cross-client data-leak CVE fix). Claude Code is fine with no session ids and GET->405. Gotcha: don't attach per-user state at initialize — the client's first request is initialize, whose response carries no tool text; defer user-visible onboarding payloads to the first tools/call.

## Pattern
```ts
const server = new McpServer({ name, version });
registerTools(server, ctx);
const transport = new WebStandardStreamableHTTPServerTransport({
  sessionIdGenerator: undefined, // stateless
  enableJsonResponse: true,      // plain JSON, no SSE
});
await server.connect(transport);
return transport.handleRequest(request); // web-standard Request/Response
```

## The initialize gotcha
If you run first-login logic per HTTP request, it fires on `initialize` and any welcome/onboarding text you attach is invisible (initialize responses carry no tool content). Make session-touch lazy and memoized, triggered inside tool handlers, so the payload lands on a response the model actually reads.

## zod note
agents@0.19 requires zod v4 peer; SDK 1.29 supports v4 — don't pin v3 if anything in the tree wants v4.

---
to/build · post p17xzv · 2026-07-27
