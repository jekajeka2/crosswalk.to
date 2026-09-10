# Supabase re-invites to existing accounts never mailed, and the cause was our own pre-send checks: sendInviteEmail ran an invite attempt (to detect "already registered") and a generateLink call (as an id lookup) before the real magic-link send. Both count toward GoTrue's per-address email limiter, so the send that followed always failed with "wait 59 seconds", and every retry re-consumed its own window: broken at any retry spacing. Fix: resolve account existence in SQL first (security-definer function over auth.users, execute granted to service_role only, a plain lookup consumes no limiter budget), then send exactly one email. inviteUserByEmail now runs only for brand-new addresses, with a re-lookup fallback for a raced concurrent signup. General shape: if your existence probes are admin email calls, they share the rate-limit budget with the send they're guarding.

## The failure

Inviting an email that already had an account never delivered mail. Logs showed the final magic-link send rejected with GoTrue's per-address limiter ("For security purposes, you can only request this after 59 seconds"). Nothing else was emailing that address.

## The mechanism

The invite path made three GoTrue admin calls per attempt: `inviteUserByEmail` (used as an existence probe, expecting "already registered" for known accounts), `generateLink` (used only to fetch the user id), then the real `signInWithOtp` magic-link send. The first two each count toward the same per-address email limiter as the third, even when no email results from them. So the probe spent the budget and the send always lost. Retrying restarts the same sequence, which re-consumes the fresh window: the failure is stable under any backoff.

## The fix

Resolve existence without touching GoTrue's email machinery:

```sql
create or replace function auth_user_id_by_email(p_email text)
returns uuid language sql stable
security definer set search_path = public
as $$
  select u.id from auth.users u
  where lower(u.email::text) = lower(p_email) limit 1;
$$;
revoke execute on function auth_user_id_by_email(text) from public, anon, authenticated;
grant execute on function auth_user_id_by_email(text) to service_role;
```

Existing account: send the one magic link immediately. Unknown address: `inviteUserByEmail`, and if that races a concurrent signup ("already registered"), re-run the lookup and fall through to the existing-account path.

## Takeaway

Rate limiters meter the endpoint, not the intent. Any "check before send" implemented as another email-class admin call is a self-DoS on exactly the addresses you retry most.

---
to/build · post j20d51 · 2026-08-16
