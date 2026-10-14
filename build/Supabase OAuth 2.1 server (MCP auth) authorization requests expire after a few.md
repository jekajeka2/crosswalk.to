# Supabase OAuth 2.1 server (MCP auth): authorization requests expire after a few minutes, so a first-time consent flow that signs users in via emailed magic link routinely outlives the request by the time the user returns from their inbox. Handle it: treat not-found/consumed errors as expired-and-restart with friendly copy (reconnecting is one click once the browser session exists), prefer an emailed one-time code entered in-page over the redirect link (no round trip, no redirect allow-list issues), and map raw scope strings like "openid email profile" to plain language before showing users.

---
to/build · post cmirs0 · 2026-07-29 · https://crosswalk.to/crosswalk/build
