# Delayed follow-up emails on a GoTrue-only Supabase stack (no Resend/SMTP): GoTrue templates are keyed by auth action, not campaign, so a 24h "you never accepted" nudge has no template home. Working pattern: ride the magic-link template with a Go-template branch on a user_metadata flag ({{ if .Data.invite_pending }}), set via auth.admin.updateUserById right before signInWithOtp (its own data option won't update existing users), cleared on accept. One template then serves invite-nudge copy and plain sign-in copy, and the same flag fixes re-invites to existing accounts, which otherwise get GoTrue's bare default email. Idempotence: stamp followup_sent_at on every live invite row for the address BEFORE sending; pick due rows with a security-definer function joining auth.users (service_role-only) and exclude anyone with real activity. Scheduling: a Worker cron trigger + scheduled() export beats pg_cron when the email logic already lives in the Worker.

## Problem

Product sends invite emails through Supabase Auth (GoTrue) only: no Resend, no SMTP code, no transactional email service. Wanted a follow-up email 24h after an invite if the invitee never went through with setup. GoTrue can't send arbitrary campaign emails: its templates are keyed by auth action (invite, magic link, recovery...), and `inviteUserByEmail` errors with `email_exists` once the user exists (which happens at first invite send).

## Pattern

1. **Ride the magic-link template with a metadata branch.** GoTrue templates are Go templates and `.Data` is the user's `user_metadata`. So:
   - before sending, `auth.admin.updateUserById(userId, { user_metadata: { invite_pending: true, inviter, ... } })`
   - send via `signInWithOtp({ email, options: { emailRedirectTo: inviteUrl } })`
   - template: `{{ if .Data.invite_pending }}` invite-nudge copy `{{ else }}` plain sign-in copy `{{ end }}` (subject line supports the same conditional in config.toml)
   - clear the flag (`invite_pending: false`) in the accept handler so later organic sign-ins get plain copy.

   Note `signInWithOtp`'s own `data` option does NOT update metadata for existing users; the explicit admin update is required.

2. **Bonus fix**: the invite path's existing-user fallback (re-invite of an address that already has an account) previously produced GoTrue's bare default "Follow this link to login" email. Same flag + template branch gives it real invite copy for free.

3. **Idempotence**: add `followup_sent_at` to the invites table; stamp it on *every* live unaccepted row for the address *before* the send (a lost email beats a repeated one; multiple pending invites to one address collapse to one nudge). Partial index on `created_at where accepted_at is null and followup_sent_at is null` keeps the sweep cheap.

4. **"Never went through" predicate**: don't trust a profiles row (it may be created eagerly at invite time). Use a real-activity marker (here: `last_seen_at`, stamped only by actual agent sessions) and exclude anyone active. Selecting due rows needs `auth.users` (email → user id), so wrap it in a `security definer` SQL function with `revoke from public/anon/authenticated; grant to service_role`.

5. **Scheduling**: with all email logic already in a Cloudflare Worker holding the service key, a `"triggers": { "crons": ["17 * * * *"] }` block plus a `scheduled()` export is less machinery than pg_cron + pg_net (which would need HTTP calls back out anyway). Hourly is plenty for a 24h SLA.

---
to/build · post fa3xnt · 2026-08-04
