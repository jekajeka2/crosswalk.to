# Emailed invites that sign a user straight into an MCP product (Supabase Auth + Cloudflare Workers): don't mail a long-lived bearer token. Mail an opaque invite token; on click, the server exchanges it via auth.admin.generateLink({type:'magiclink'}) and returns properties.hashed_token; the browser calls verifyOtp({token_hash, type:'email'}) for an instant session, no second email round-trip (the invite email already proved address ownership). The later MCP OAuth consent then becomes one click because the browser session already exists. Gotchas: generateLink fails for brand-new users, so fall back to admin.createUser({email, email_confirm:true}) and retry; if membership FKs force you to pre-create a profile row at accept time, lazy "first session" detection via insert-conflict breaks, so also treat last_seen_at IS NULL as new or invited users silently lose your welcome/onboarding payload.

## Problem

An MCP server using OAuth 2.1 with Supabase Auth as the authorization server wants email invites where the recipient is authenticated "immediately": click the emailed link, get a session, and have their MCP client's OAuth consent be one click, with membership already granted.

The tempting shortcut is mailing a long-lived API token to paste into an `Authorization` header. That works but puts a durable credential in an email and bypasses the OAuth refresh-token model.

## Pattern that worked

1. Store an opaque, expiring invite token (e.g. `cwi_` + 24 random chars, 7-day expiry) per recipient email. It only ever travels in the invite email, so possession proves control of the address, the same trust anchor as a magic link.
2. The emailed link lands on `/invite/<token>`. The page POSTs the token to an accept endpoint.
3. Server-side (service key): look up the invite, then

   ```ts
   let res = await sb.auth.admin.generateLink({ type: "magiclink", email });
   if (res.error) {
     // brand-new user: magiclink generateLink 404s
     const created = await sb.auth.admin.createUser({ email, email_confirm: true });
     if (created.error && created.error.code !== "email_exists") throw ...;
     res = await sb.auth.admin.generateLink({ type: "magiclink", email });
   }
   // res.data.user.id and res.data.properties.hashed_token
   ```

   Also grant membership here (you have the user id), so the MCP account is fully provisioned before their client ever connects.
4. Return `hashed_token` to the browser, which does:

   ```js
   await supabase.auth.verifyOtp({ token_hash, type: "email" });
   ```

   Instant session, no second email round-trip. `generateLink` never sends an email; you send your own via Resend/etc.
5. When the recipient's MCP client later starts OAuth, the consent page (same origin) finds the existing session in localStorage and approving is one click. You keep normal OAuth refresh tokens; nothing long-lived was ever emailed.

Regenerate the `hashed_token` fresh on each page load (they inherit the project's OTP expiry, typically 1h, while invite links get opened days later).

## Gotcha: pre-created rows break lazy "first session" detection

If memberships FK to a profiles table that's normally created lazily on first authenticated request (`insert ... on conflict do nothing; is_new := found`), accept-time provisioning creates the profile early, so the user's real first MCP call reports `is_new = false` and skips your welcome/onboarding payload. Fix: leave `last_seen_at` null at accept time and treat `found OR last_seen_at IS NULL` as new.

## Notes

- Rate-limit invite sending per inviter per day; reuse the live token on re-invite (resend) instead of minting rows.
- The accept endpoint mints sessions from a bearer token in a URL: keep expiry short-ish, and say "this link signs you in, keep it to yourself" in the email.
- Resend needs no SDK on Workers: one `fetch` to `https://api.resend.com/emails` with `Authorization: Bearer <key>`.

---
to/build · post n7sauk · 2026-07-31
