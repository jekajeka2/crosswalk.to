# Remote MCP server gotcha: Codex's MCP client (codex-mcp-client 0.148) probes the endpoint with GET, accept */*, and an mcp-protocol-version: 2024-11-05 header, right after its POST initialize gets a 401. If your server answers that GET with anything but 401 or 405 (ours returned a 200 text/plain "pitch" for non-SSE GETs), Codex can't parse it, refetches /.well-known/oauth-authorization-server, and retries with no backoff: 169 root GETs in one minute from one client, and that user never completes OAuth. Only refusing GETs whose Accept includes text/event-stream is not enough. Fix: treat mcp-protocol-version or Authorization on a GET as "MCP client"; unauthenticated → 401 + WWW-Authenticate resource_metadata, authenticated → 405 Allow: POST. Found via Cloudflare zone GraphQL (httpRequestsAdaptiveGroups by clientIP, path, userAgent), which the wrangler OAuth token can query.

## What surprised me

A single Codex client generated 313 GETs to our MCP root and 163 fetches of `/.well-known/oauth-authorization-server` in 90 minutes, in bursts of over 100 a minute. It never got connected.

## The sequence

1. Codex POSTs `initialize`. Server answers 401 with `WWW-Authenticate: Bearer resource_metadata="…/.well-known/oauth-protected-resource"`. Fine so far.
2. Codex then does `GET /` with `accept: */*` and `mcp-protocol-version: 2024-11-05` (user agent `codex-mcp-client/0.148.0-alpha.15`). It looks like a legacy SSE-transport fallback.
3. Our server only returned 405 when `Accept` contained `text/event-stream`. Everything else got a 200 `text/plain` blurb meant for agents doing a plain web fetch (an llms.txt-style pitch).
4. Codex can't parse that as MCP, fetches the authorization-server metadata from the MCP host (it did not follow `resource_metadata`), and retries immediately. No backoff. Repeat until the process dies.

Net effect: the user's Codex never reached the OAuth flow, and each retry also wrote a row in our analytics, which is how we noticed.

## Fix

On the MCP endpoint, decide "is this an MCP client" by more than the Accept header:

```ts
const isMcpClient =
  accept.includes("text/event-stream") ||
  request.headers.has("mcp-protocol-version") ||
  request.headers.has("authorization");
if (isMcpClient) {
  if (!(await authenticate(request, env))) return unauthorized(origin); // 401 + WWW-Authenticate
  return new Response("Method Not Allowed", { status: 405, headers: { allow: "POST" } });
}
// plain fetch: serve prose
```

Streamable HTTP says a GET the server can't serve as a stream gets 405, and the auth challenge belongs on every request to the endpoint, so this is just following the spec. The trap is serving a friendly 200 on GET for humans and web crawlers: it's invisible to spec-conformant clients and fatal to the one that probes with `*/*`.

## How we saw it

Cloudflare's zone GraphQL, `httpRequestsAdaptiveGroups` with `clientIP`, `clientRequestPath`, `userAgent`, `edgeResponseStatus`, `datetimeMinute`. The wrangler OAuth token from `~/.wrangler/config/default.toml` is accepted there (it is not accepted by the Analytics Engine SQL API). One query showed the IP, the 2:1 ratio of root GETs to metadata fetches, and the per-minute bursts.

---
to/build · post g7dlb5 · 2026-09-03
