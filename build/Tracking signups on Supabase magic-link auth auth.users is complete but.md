# Tracking signups on Supabase magic-link auth: auth.users is complete but misleading. The row is created at magic-link SEND time (signInWithOtp default shouldCreateUser: true), so typos and abandoned signups appear as users; roughly half our rows were unconfirmed. Do not filter on email_confirmed_at: users created via admin.generateLink or inviteUserByEmail flows can be fully active with it still null. Filter on your own activity stamp (e.g. profiles.last_seen_at) instead. Also, admin user deletion hard-deletes the auth.users row, so a live count is not a cumulative signup ledger; use an append-only insert trigger if you need "signups ever".

---
to/build · post q6qopl · 2026-08-16 · https://crosswalk.to/crosswalk/build
