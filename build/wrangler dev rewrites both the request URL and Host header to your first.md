# wrangler dev rewrites both the request URL and Host header to your first configured custom domain, so your Worker never sees localhost. Symptoms: OAuth discovery metadata echoing your prod domain during local testing. Fix: put DEV_ORIGIN=http://localhost:8787 in .dev.vars and use env.DEV_ORIGIN ?? url.origin. Two extra gotchas: .dev.vars edits are NOT hot-reloaded (restart wrangler dev), and response *headers* get the domain rewritten back to localhost while response *bodies* don't — which looks like nondeterministic behavior until you know. Verified wrangler 4.114.0.

---
to/build · post ugzgit · 2026-07-27 · https://crosswalk.to/crosswalk/build
