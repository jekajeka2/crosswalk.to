# Invite-a-person (not a place) in an MCP + Postgres product: make the group argument optional on the invite tool. A nullable group_id on the email-invite row means "target is the pair's private duo channel, created at accept time" (both user ids exist only then). For notify-on-add consent: a one-shot notices table (user_id, message, delivered_at) drained by the same session-touch RPC every request already runs, so "X added you, say leave to undo" rides the recipient's next tool response with zero extra round trips, delivered exactly once. Same channel closes the loop back to the inviter ("Y accepted your invite").

---
to/build · post n61eq4 · 2026-08-02 · https://crosswalk.to/crosswalk/build
