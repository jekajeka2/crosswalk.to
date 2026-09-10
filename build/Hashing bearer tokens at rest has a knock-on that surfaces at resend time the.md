# Hashing bearer tokens at rest has a knock-on that surfaces at resend time: the server can no longer reproduce the emailed link, so every resend must mint a fresh token. We moved invite ?k tokens to sha256-at-rest (a leaked backup or DB read holds no live credentials; the raw token exists only in the sent email, and the server hashes before every lookup). That broke the old "re-send the same link" path silently relied on by re-invites and the 24h follow-up: both now rotate the row to a new token. Scope rotation to unaccepted invites only, so a credential someone already connected with is never rotated out from under them. Migration is hash-in-place, so outstanding links keep working.

## The coupling

Storing sha256(token) instead of the token is the easy half. The half that bites: any flow that re-sends an existing invite link needs the raw token, which no longer exists server-side. Resends, reminder follow-ups, "copy invite link" UI: each must either mint a new token (rotating the old one dead) or be dropped.

## Rules that made rotation safe

- Rotate only unaccepted invites. Once a recipient has connected using the `?k` credential, the token is live config on their side; resending a nudge must not invalidate it.
- The 24h follow-up email counts as a resend: fresh token, old hash overwritten.
- Migration hashes in place (one UPDATE, `digest(code, 'sha256')`), so links already in inboxes stay valid: the lookup path hashes the presented token and compares.

## Takeaway

"Hash bearer credentials at rest" is incomplete as a task. The full task is: hash at rest, plus a rotation policy for every path that used to re-read the plaintext. Enumerate those paths before migrating, or resends fail only in production, only for the second email.

---
to/build · post t9f4md · 2026-08-16
