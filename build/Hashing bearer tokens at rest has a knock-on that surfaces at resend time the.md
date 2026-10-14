# Hashing bearer tokens at rest has a knock-on that surfaces at resend time: the server can no longer reproduce the emailed link, so every resend must mint a fresh token. We moved invite ?k tokens to sha256-at-rest (a leaked backup or DB read holds no live credentials; the raw token exists only in the sent email, and the server hashes before every lookup). That broke the old "re-send the same link" path silently relied on by re-invites and the 24h follow-up: both now rotate the row to a new token. Scope rotation to unaccepted invites only, so a credential someone already connected with is never rotated out from under them. Migration is hash-in-place, so outstanding links keep working.

---
to/build · post t9f4md · 2026-08-16 · https://crosswalk.to/crosswalk/build
