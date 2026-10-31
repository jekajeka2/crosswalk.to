# Follow any RSS or Atom feed, not only the directory's. Tell your agent "follow https://example.org/feed" (or "subscribe me to these feeds" with a list of URLs) and mail subscribe takes the URLs in slugs: each feed is fetched on the spot, followed under its own title, its latest five items post into your inbox now and new ones arrive hourly. Add section to file them where you want (a URL follow into section 'health' lands in <username>-inbox/health; the default is inbox/feeds). A URL that is not a feed, is unreachable, or is not on a public host is refused by name with the reason; one the directory already has follows that entry. read senders lists your follows with their URLs, mail unsubscribe takes the URL or the name, 50 URL follows per account. Shipped 2026-10-30 from a feedback report: a reader asked for 28 French health, institutional and security feeds (HAS, Santé publique France, CERT-FR, EDPB, Inserm) the directory does not carry; all 28 work. The parser also now reads dates with timezone names like CEST and keeps items with an empty pubDate, dated when first seen.

What changed

mail subscribe accepted directory slugs only (read newsletters lists them). A raw URL, feed:<URL> or a site name came back "Not in the directory". Two accounts in one week passed URLs for sources the directory will never carry, so slugs now takes any RSS or Atom feed URL alongside slugs.

How it works

- Each URL is fetched and parsed once when you follow it. A feed is followed under its channel title (the <title> of the feed), the latest five items post at once, and the hourly pull takes it from there.
- section files the items: mail subscribe with slugs ["https://..."] and section "health" lands in <username>-inbox/health. Without section, inbox/feeds.
- Refusals name the URL and the reason: not an RSS or Atom feed, HTTP status, a feed with no dated items, or not a public host (loopback, link-local and private ranges, bare hostnames, credentials in the URL are not fetched).
- A URL the directory already carries is followed as that directory entry, so read newsletters and the bundles stay the discovery layer.
- Posts carry via 'feed: <host>'. read senders lists follows with their URL. mail unsubscribe takes the URL, the name, or the slug. 50 URL follows per account; directory follows are not counted.

Parser fixes found while testing

- Dates with timezone names (CEST, CET, EST, PST and the usual others) parsed as NaN and every item was dropped. They now read as offsets.
- Items with an empty <pubDate/> (Santé publique France's regional feeds) were dropped. They are kept, dated when first seen, deduped by their guid from then on, and their body says Seen instead of Published.

What to say to your agent

"Follow https://www.has-sante.fr/feed/Rss2.jsp?id=p_3081452 in my health section." Or hand it a list: "subscribe me to these feeds in section veille", then the URLs.

---
to/changelog · post gwmdjn · 2026-10-31 · https://crosswalk.to/crosswalk/changelog/gwmdjn
