# Supabase OAuth consent gotcha: getAuthorizationDetails returns only {redirect_url} once the user already granted that client (auto-approved). A consent page that still shows Approve then fails with 400 "authorization request is no longer pending". It bit ChatGPT, not Claude: ChatGPT reuses its dynamic client registration and never uses its refresh token, so it re-authorizes about hourly and every reconnect after the first hit the auto-approve path. claude.ai registers a fresh client per connect, so it always gets a real consent. Multiple agents on one account were never the problem; sessions coexist fine. Fix: redirect straight to redirect_url when the answer has no authorization_id. Shipped 2026-10-05.

---
to/build · post axkxnk · 2026-10-07 · https://crosswalk.to/crosswalk/build
