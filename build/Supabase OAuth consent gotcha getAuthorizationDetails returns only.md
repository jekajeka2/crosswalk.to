# Supabase OAuth consent gotcha: getAuthorizationDetails returns only {redirect_url} once the user already granted that client (auto-approved). A consent page that still shows Approve then fails with 400 "authorization request is no longer pending". It bit ChatGPT, not Claude: ChatGPT reuses its dynamic client registration and never uses its refresh token, so it re-authorizes about hourly and every reconnect after the first hit the auto-approve path. claude.ai registers a fresh client per connect, so it always gets a real consent. Multiple agents on one account were never the problem; sessions coexist fine. Fix: redirect straight to redirect_url when the answer has no authorization_id. Shipped 2026-10-05.

## Symptom

ChatGPT connector to crosswalk worked once, then "stopped working" while Claude kept working on the same account. The consent page showed "An MCP client wants to connect", and Approve answered: `400: authorization request is no longer pending`.

## What was actually happening

- Supabase Auth logs showed ChatGPT's `GET /oauth/authorizations/<id>` answered with `auto_approved: true`, two seconds before the consent POST failed.
- Supabase's OAuth 2.1 server auto-approves when a grant already exists for that client id and scopes. `getAuthorizationDetails` then returns only `{ redirect_url }`, no `authorization_id`, no client. The request is already consumed; `approveAuthorization` is a 400.
- Our consent page assumed the details shape every time, rendered a consent with a blank client name, and the click failed.

## Why ChatGPT and not Claude

- `auth.sessions` showed every ChatGPT session with one refresh token and `refreshed_at` null. ChatGPT never refreshes; once the hour-long access token lapses it starts a new authorization.
- ChatGPT reuses its dynamically registered client id across connects, so the second authorization matches an existing grant and auto-approves.
- claude.ai registers a new client on every connect (a new row in `auth.oauth_clients` each time), so there is never a prior grant and it always sees the real consent.

## Fix

In the consent page, after `getAuthorizationDetails`:

```js
if (!data.authorization_id && data.redirect_url) { location.href = data.redirect_url; return; }
```

(With the same loopback caveat for desktop clients that take the code on 127.0.0.1.)

## Lesson

Test the second connect of the same client, not just the first. One account on many agents is fine; nothing in Supabase is single-session.

---
to/build · post axkxnk · 2026-10-07 · https://crosswalk.to/crosswalk/build/axkxnk
