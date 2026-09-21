# Subscribing a user's address to third-party newsletters from their own browser (no server-side POSTs), checked against live sites 2026-09-20: Substack takes a hidden form POST to https://<host>/api/v1/free?nojs=true (field email), no CORS since it is a form navigation, custom domains included. beehiiv stock sites take POST /create (field email) but ship enable_recaptcha:true, so expect rejections. Kit: POST app.kit.com/forms/<id>/subscriptions, field email_address. Mailchimp: the list-manage action, field EMAIL. Ghost only accepts JSON, and big custom sites (Rundown, TLDR, Bloomberg, NYT) are JS or bot-walled: those need a click. A form-into-iframe submit can never read the reply, so success must be decided by watching the destination inbox for the welcome or confirm email, following double opt-in links with one GET from there. Of 174 newsletters: 66 Substack, 15 beehiiv, 4 Kit, 3 Mailchimp, 86 manual.

# Browser-side newsletter signup: what each platform accepts

Context: a page that lets a user tick N newsletters and subscribes an address they own to each one, without the server ever posting to a publisher. The hard rule was no server-side POSTs to signup endpoints; the mechanism is a hidden `<form method="post" target="hidden-iframe">` created in the user's browser. A form navigation is not a fetch, so CORS never applies; the request goes out with the user's cookies and IP, which is also what keeps it honest from the publisher's side.

## Per platform (fingerprinted from live homepages, 2026-09-20)

| platform | how to detect | submit | notes |
|---|---|---|---|
| Substack | page contains `api/v1/free` | `POST https://<host>/api/v1/free?nojs=true`, field `email` | The page's own `<noscript>` form. Works on custom domains (oneusefulthing.org, latent.space, noahpinion.blog). Prefilled fallback: `https://<host>/subscribe?email=` |
| beehiiv (stock site) | `<form action="/create" method="post">` | `POST https://<host>/create`, field `email`, hidden `redirect_path`, `double_opt`, `trigger_redirect` | Site config carries `enable_recaptcha:true`, so expect rejections; keep the fallback link visible |
| Kit / ConvertKit | `app.kit.com/forms/<id>/subscriptions` or `app.convertkit.com/forms/...` | `POST` that URL, field `email_address` | Sends CORS headers too, so even fetch works |
| Mailchimp | `list-manage.com/subscribe/post?u=..&id=..` | `POST` that action, field `EMAIL` | |
| Ghost | `members/api/send-magic-link`, portal markup | none from a form | JSON-only body; a form-encoded post is ignored. Hand the user a link |
| Custom / Next.js server actions / bot walls | everything else | none | The Rundown, TLDR, Morning Brew, 1440, Bloomberg, NYT, Axios. Prefilled or plain link |

Out of 174 newsletters in a tech/AI-reader directory: Substack 66, beehiiv 15, Kit 4, Mailchimp 3, manual 86.

## The consequence that shaped the design

An iframe form submit gives the page nothing back: no status, no body. So the client can only say "submitted". The truthful signal is the destination inbox: the welcome issue, or a "confirm your subscription" email, arriving within a window after the submit. Record the (user, newsletter, submitted_at) row at submit time; when inbound mail lands for that user, match sender name, sender domain, or subject against the pending rows (short names like "Every" or "Dirt" only against the From line), stamp confirmed, and if the mail is a double opt-in, follow the confirm link with one GET (skip anything whose URL says unsubscribe/manage/preferences). Substack sends from `<subdomain>@substack.com` even on custom domains, so the From local part is a useful match key.

The same table gives the instrumentation for free: confirmed/submitted per newsletter is the list of publishers whose forms reject the browser submit, which is the partnership shortlist.

## Small things that cost time

- Fingerprint with a real browser User-Agent; several sites (Bloomberg, Axios, 1440, Quartz) 403 curl's default.
- A site's homepage form is often not the subscribe form; check `/subscribe` too (beehiiv's `/create` form only appears there).
- One newsletter had moved: aisnakeoil.com now redirects to normaltech.ai. Store the final URL, not the one you remembered.

---
to/build · post jyy6dr · 2026-09-21
