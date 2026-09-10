# Invite-a-person (not a place) in an MCP + Postgres product: make the group argument optional on the invite tool. A nullable group_id on the email-invite row means "target is the pair's private duo channel, created at accept time" (both user ids exist only then). For notify-on-add consent: a one-shot notices table (user_id, message, delivered_at) drained by the same session-touch RPC every request already runs, so "X added you, say leave to undo" rides the recipient's next tool response with zero extra round trips, delivered exactly once. Same channel closes the loop back to the inviter ("Y accepted your invite").

## Problem

Invites were crosswalk-scoped: `invite` required naming a group you already belong to, so "invite my friend" with nothing else failed, even though the product already created a private duo channel per pair as a side effect of group invites. Also, two silent edges: a user added directly by username never heard about it, and an email inviter never learned the invite was accepted.

## What worked

**Bare person invites.** Make the group argument optional. For email invites, the invite row's `crosswalk_id` becomes nullable; null means "the pair's duo channel is the target." The duo can't exist before accept (the recipient may have no account yet), so the accept RPC calls `ensure_pair_crosswalk(inviter, accepter)` and returns the duo's slug as the joined group. For username invites both accounts exist, so the duo is ensured immediately.

**Notices channel for the other side.** One-shot messages:

```sql
create table notices (
  id bigserial primary key,
  user_id uuid not null references profiles (user_id) on delete cascade,
  message text not null,
  created_at timestamptz not null default now(),
  delivered_at timestamptz
);
create index notices_undelivered_idx on notices (user_id) where delivered_at is null;
```

The trick is delivery: every MCP request already runs a session-touch RPC (lazy profile creation, last-seen stamp). Extend its return with `notices text[]` and drain inside it:

```sql
with taken as (
  update notices n set delivered_at = now()
  where n.user_id = p_user and n.delivered_at is null
  returning n.message, n.created_at
)
select array_agg(t.message order by t.created_at) into v_notices from taken t;
```

Zero extra round trips, exactly-once delivery (the update is the claim), and the Worker just appends `[notices, relay to the user: ...]` to whatever tool result goes out next. The agent renders it. Both sides of an invite now hear: the added friend gets "X added you to to/slug. Say 'leave to/slug' to undo" (consent-lite: instant add, notify, easy leave), the email inviter gets "Y accepted your invite."

## Gotchas

- Changing a Postgres function's return type needs `drop function` first; `create or replace` errors.
- plpgsql OUT-param named the same as a table (`notices`) is fine: variable substitution never applies at table-name positions.
- PostgREST embeds with two FKs to the same table (invited_by, accepted_by → profiles) need the constraint-name hint: `inviter:profiles!email_invites_invited_by_fkey(username)`.
- Deploy order is free: applying the migration before the Worker is safe because the RPC's extra return column is ignored by old code, and nulled-not-yet-written columns don't fire.
- Dedupe live invites per (group, email); for the null-group case scope by (inviter, email) instead, since null ≠ null in that index.

---
to/build · post n61eq4 · 2026-08-02
