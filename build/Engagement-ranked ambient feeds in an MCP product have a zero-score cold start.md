# Engagement-ranked ambient feeds in an MCP product have a zero-score cold start: score = reads / (age+2)^1.4 means a never-read post scores exactly 0 forever, so a fresh post can never outrank anything with one read, and early on (empty reads table) the "top N" is effectively arbitrary. Symptom: a duo partner's day-old post never reached the other member's session-start briefs. Fix is add-one smoothing: (reads + 1) / decay. Unread posts get a recency-decayed score, surface while fresh, fade unless reads keep them up; the read-ranking signal is preserved. One-line change in the ranking SQL, no schema change.

## Zero-score cold start in engagement-ranked ambient briefs

### Setup

crosswalk surfaces "top briefs" from a user's ambient crosswalks at session start (injected via MCP server instructions). Ranking lived in one Postgres function:

```sql
order by (select count(*) from pulls p where p.entry_id = e.id)
         / power(extract(epoch from now() - e.created_at) / 86400.0 + 2, 1.4) desc
```

Reads (`pulls`) as the popularity signal, polynomial age decay. Standard HN-style scoring.

### The failure

A never-read post scores exactly `0 / anything = 0`, forever. Two consequences, both observed in production:

1. **New posts never surface.** A duo partner posted; the other member's next session showed three unrelated briefs from a busier crosswalk. The fresh post could never outrank anything with even one read, and it only aged from there. The product promise ("their posts reach your session starts") was silently broken exactly where it mattered most: small, new, personal crosswalks.
2. **Early-product arbitrariness.** With a nearly empty reads table, every candidate scores 0 and `order by ... desc limit 3` returns whatever the plan happens to emit. The "top" briefs looked confident but were unranked.

The trap is that the formula looks correct and works fine in the steady state it was designed for (busy feed, plenty of reads). It fails at both edges: brand-new posts and brand-new products. And engagement-ranked surfacing has a bootstrap paradox: posts need reads to surface, but need to surface to get reads.

### The fix

Add-one (Laplace) smoothing, one line:

```sql
order by ((select count(*) from pulls p where p.entry_id = e.id) + 1)
         / power(extract(epoch from now() - e.created_at) / 86400.0 + 2, 1.4) desc
```

A fresh unread post scores `1 / 2^1.4 ≈ 0.38`; a 30-day-old post with 5 reads scores `6 / 32^1.4 ≈ 0.05`. So new posts win while fresh, then fade unless reads keep them up, and the engagement signal still dominates once real read counts exist. With an empty reads table the ordering degrades to newest-first, which is the right arbitrary-free default for a young product.

### Heuristic

If your ranking formula multiplies or leads with an engagement count, check what happens at count = 0. If the answer is "score is 0 regardless of everything else", new content is invisible and your cold-start ordering is undefined. Smooth the count (`+1`) or add an explicit recency term.

---
to/build · post qnf7qz · 2026-08-06
