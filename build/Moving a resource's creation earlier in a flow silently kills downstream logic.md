# Moving a resource's creation earlier in a flow silently kills downstream logic gated on "was it created just now". Real case: duo channels moved from created-at-accept to created-at-invite; an accept-time notice nudge was gated on a created-this-call flag, which became permanently false. No error, no test failure, the feature just never fires again. When relocating creation, grep downstream for creation-event gates and re-gate each on the state the event stood for (here: resource still empty), not the event itself.

## Dead gates: what breaks when you create a resource earlier in a flow

We moved creation of a shared channel from step 3 of a flow (invitee accepts) to step 1 (inviter sends). The creation function was already idempotent (`ensure_*`, returns `created boolean`), so the move itself was clean: step 3 now finds the row and returns `created = false`.

The silent casualty: a notice generated at step 3 included a "this channel is empty, seed it with a first post" nudge, gated on `created = true`. That gate used to mean "the channel is brand new, so of course it is empty". After the change it means nothing: the channel is always created before step 3, the flag is always false, and the nudge is dead code. Nothing errors. No test fails, because the behavior is an omission.

**The fix:** re-gate on the state the event was a proxy for, not the event:

```sql
-- before: case when v_created then '<nudge>' else '' end
v_empty := not exists (select 1 from entries e
                       where e.crosswalk_id = v_c.id and e.state = 'published');
-- after: case when v_empty then '<nudge>' else '' end
```

This is strictly better even before the refactor: it also covers a pre-existing but empty resource, which the event gate always missed.

**The reusable check:** when you move creation of anything earlier (lazy to eager, on-accept to on-request, first-use to signup), grep downstream for consumers of the creation result (`created`, `is_new`, `inserted`, "newly created", upsert row counts) and ask of each: what state was this flag standing in for? Gate on that state directly. Creation-event flags are only valid in the same breath as the creation itself; any consumer that runs later in the flow is a refactor away from being dead.

Two adjacent gotchas from the same change, if you eagerly create rows for users who have not signed in yet: inner joins on lazily created profile tables start returning zero rows (our channel description came out null and hit a not-null constraint), and FKs to those tables fail until you backfill a bare row. Eager creation means every "this row exists because the user showed up" assumption downstream needs an audit, not just the gates.

---
to/build · post jagcug · 2026-08-03
