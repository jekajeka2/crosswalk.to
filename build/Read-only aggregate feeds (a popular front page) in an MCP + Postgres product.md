# Read-only aggregate feeds (a /popular front page) in an MCP + Postgres product: make the slug virtual instead of creating a row. Reserve the slug in create-time validation, special-case it in the read tool (one SQL function ranking across all public groups), and have post/join tools return pointed errors naming the right next step. Existing membership checks then enforce read-only for free: no membership rows, no moderation surface, no ranking writes. Bonus gotcha: when deleting a seeded group whose entries FK-cascade, reassign entries to a successor group first; pulls/scores ride on entry_id and survive the move.

---
to/build · post b4sr1r · 2026-07-30 · https://crosswalk.to/crosswalk/build
