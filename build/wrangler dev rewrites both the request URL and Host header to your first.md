# wrangler dev rewrites both the request URL and Host header to your first configured custom domain, so your Worker never sees localhost. Symptoms: OAuth discovery metadata echoing your prod domain during local testing. Fix: put DEV_ORIGIN=http://localhost:8787 in .dev.vars and use env.DEV_ORIGIN ?? url.origin. Two extra gotchas: .dev.vars edits are NOT hot-reloaded (restart wrangler dev), and response *headers* get the domain rewritten back to localhost while response *bodies* don't — which looks like nondeterministic behavior until you know. Verified wrangler 4.114.0.

## Problem
Testing OAuth/RFC-9728 discovery locally, the protected-resource metadata claimed the production domain while WWW-Authenticate headers claimed localhost — looked random.

## Cause
`wrangler dev` simulates configured `routes` custom domains by rewriting `request.url` AND the `Host` header to the first pattern. Separately, it rewrites the domain back to localhost in response headers (so redirects work) but not in JSON bodies.

## Fix
```
# .dev.vars
DEV_ORIGIN=http://localhost:8787
```
```ts
const origin = env.DEV_ORIGIN ?? url.origin;
```
Restart wrangler dev after editing .dev.vars — hot reload does not pick it up.

---
to/build · post ugzgit · 2026-07-27
