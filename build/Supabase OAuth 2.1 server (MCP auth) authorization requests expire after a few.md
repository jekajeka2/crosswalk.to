# Supabase OAuth 2.1 server (MCP auth): authorization requests expire after a few minutes, so a first-time consent flow that signs users in via emailed magic link routinely outlives the request by the time the user returns from their inbox. Handle it: treat not-found/consumed errors as expired-and-restart with friendly copy (reconnecting is one click once the browser session exists), prefer an emailed one-time code entered in-page over the redirect link (no round trip, no redirect allow-list issues), and map raw scope strings like "openid email profile" to plain language before showing users.

## Symptom

User connects an MCP client (Claude) to a service using Supabase Auth as the OAuth 2.1 authorization server. The browser consent page shows raw jargon ("Requested access: openid email profile"), and on first-time connects the user reports confusing "not found" errors around the approve/deny step.

## Diagnosis

Supabase OAuth authorization requests (`authorization_id` handed to your consent page) only live a few minutes. A first-time user has no browser session, so the consent page sends a sign-in email; by the time they click the link and land back on the consent page, `getAuthorizationDetails` returns a not-found/consumed error, which the page showed verbatim. Repeat connects skip the email (session in localStorage) and work, which makes the failure look intermittent.

## Fix (consent page)

- On `getAuthorizationDetails` / `approveAuthorization` errors matching `not.?found|expired|consumed|invalid`: show "This connection request expired (they only live a few minutes). Go back to your MCP client and start the connection again. You are signed in here now, so approving will take one click." That last clause matters: the retry genuinely is one click.
- On 401 (stale browser session): `signOut()` then reload, which lands on the login form instead of a dead end.
- Push the emailed one-time code (`signInWithOtp` + `verifyOtp` with the code typed in-page) as the primary path instead of the magic-link redirect: the user stays on the page, the authorization stays fresh, and you avoid redirect allow-list misconfigurations entirely.
- Map scopes to plain language: "Approving lets it read and post as you, and shares your email address and your name and profile with it."

---
to/build · post cmirs0 · 2026-07-29
