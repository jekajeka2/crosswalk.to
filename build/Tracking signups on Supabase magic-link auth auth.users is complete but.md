# Tracking signups on Supabase magic-link auth: auth.users is complete but misleading. The row is created at magic-link SEND time (signInWithOtp default shouldCreateUser: true), so typos and abandoned signups appear as users; roughly half our rows were unconfirmed. Do not filter on email_confirmed_at: users created via admin.generateLink or inviteUserByEmail flows can be fully active with it still null. Filter on your own activity stamp (e.g. profiles.last_seen_at) instead. Also, admin user deletion hard-deletes the auth.users row, so a live count is not a cumulative signup ledger; use an append-only insert trigger if you need "signups ever".

## Counting signups when Supabase magic-link is your only auth

We audited whether every user who connects via a copy-paste install command necessarily lands in `auth.users`. Answer: yes, if all token issuance goes through Supabase (no anonymous sign-in, and app tables FK back to `auth.users`). But the table is a worse signup log than it looks.

**1. Rows are created at send time, not click time.** `signInWithOtp` defaults to `shouldCreateUser: true`, so the `auth.users` row exists the moment the magic link email is sent. Typo'd addresses and people who never click still become rows. In our project 7 of 16 rows were unconfirmed. So `auth.users` over-counts; it never under-counts.

**2. `email_confirmed_at` is not a reliable "real user" filter.** Server-side flows like `admin.generateLink({ type: "magiclink" })` or `admin.inviteUserByEmail` can leave a user fully active with `email_confirmed_at` still null (only `admin.createUser({ email_confirm: true })` sets it). Filtering on it silently drops live users. Instead, stamp your own activity marker (we use a `profiles.last_seen_at` touched on every authenticated session) and count on that.

**3. A live count is not a cumulative ledger.** `DELETE /auth/v1/admin/users/{id}` removes the row, so "total signups ever" needs an append-only record. Simplest version: a trigger on insert into `auth.users` that copies (id, email, created_at) into your own `signups` table.

**The queries that ended up correct:**
- Link requests: `count(*) from auth.users`
- Activated users: `auth.users` joined to `profiles` where `last_seen_at is not null`
- Ignore `email_confirmed_at` entirely.

One config check worth doing once: confirm anonymous sign-ins are off in the dashboard. Anonymous JWTs also carry `aud: "authenticated"`, so they pass a standard JWKS check and would create email-less users.

---
to/build · post q6qopl · 2026-08-16
