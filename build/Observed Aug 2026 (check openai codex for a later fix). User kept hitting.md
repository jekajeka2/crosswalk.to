# Observed Aug 2026 (check openai/codex for a later fix). "User kept hitting 127.0.0.1 error pages during MCP OAuth connect, then it suddenly worked" — the server was innocent. Codex desktop starts TWO OAuth login flows per connect attempt: the fingerprint is paired "Codex" dynamic client registrations ~1s apart on adjacent loopback ports in your AS's client registry (Codex CLI registers once). Its loopback listener dies the instant a flow is dropped (Drop guard calls server.unblock() in codex-rs rmcp-client), so approving the superseded flow's tab hits a dead 127.0.0.1 port. Retries re-roll the race. Ruled out: the rmcp RFC 9207 iss bug (needs the AS to advertise/send iss — Supabase GoTrue does neither), pending-authorization expiry (GoTrue default 10m), redirect allowlists (DCR clients validate against their own URIs), and Codex's 300s callback timeout.

> **Observed August 2026** against the Codex desktop build current at the time (source references are to `openai/codex` main as of then). If you're reading this much later, check whether the double-flow behavior has since been fixed upstream before acting on it.

## Symptom

A new user connecting to our remote MCP server from **Codex desktop** kept landing on browser error pages at `http://127.0.0.1:<port>/...` during the OAuth sign-in, then on the third try it worked with no changes anywhere. iOS/CLI Codex setups had been flawless.

## The fingerprint that cracked it

The authorization server's OAuth client registry (we use Supabase's OAuth 2.1 server; any AS with dynamic client registration will show the same) had **six** "Codex" registrations from the session — **three pairs, each pair ~1 second apart on adjacent loopback ports** (e.g. `127.0.0.1:53271` and `:53285`), all with the same per-install callback suffix. A successful Codex CLI setup from weeks earlier had exactly **one** registration.

So: desktop Codex was starting **two OAuth login flows per connect attempt**. Each flow registers its own client and binds its own loopback listener.

## Why that produces dead 127.0.0.1 pages

In `codex-rs/rmcp-client/src/perform_oauth_login.rs`, the callback server is held by a `CallbackServerGuard` whose `Drop` impl calls `server.unblock()` — the listener dies the instant a login-flow future is dropped. When a second flow supersedes the first, the first flow's port goes dead while its authorize URL may still be sitting in a browser tab. Approve that tab → the AS redirects to the dead port → "127.0.0.1 refused to connect". Retrying re-rolls the race; it "eventually works" when the tab you approve belongs to the still-live listener.

## What we ruled out (useful checklist for the same symptom)

- **The known Codex/rmcp RFC 9207 bug** (`"missing required issuer"`, openai/codex #31573/#32472/#34684): only fires when the AS advertises `authorization_response_iss_parameter_supported` or sends `iss` in the callback. Supabase GoTrue does neither (`buildSuccessRedirectURL` appends only `code` and `state`), so it can't be this — check your AS metadata before chasing it.
- **Pending-authorization expiry**: GoTrue's default authorization TTL is 10 minutes (`OAUTH_SERVER_AUTHORIZATION_TTL`, default `10m`), auth codes 10 minutes. A multi-minute consent (e.g. magic-link email sign-in mid-flow) still fits.
- **Codex's own callback timeout**: 300s (`DEFAULT_OAUTH_TIMEOUT_SECS`), so slow consent within one attempt isn't it either.
- **Redirect-URI allowlists**: DCR clients are validated against their own registered URIs; every `127.0.0.1:<port>` registration was accepted.

## Takeaways if you run a remote MCP server

- Browser errors at `127.0.0.1` during OAuth are almost never your server — that's the client's loopback callback hop. First move: read your AS's client registry and count registrations per attempt.
- Consider softening the post-consent redirect: instead of instantly `location.href`-ing to the client's loopback callback, show "Approved — returning you to <app>… if the next page can't connect, go back to the app and retry sign-in" for a beat. When the loopback hop fails, the user at least knows the fix is client-side retry, not your server.
- Tell users to approve promptly and avoid double-clicking connect: each fresh attempt kills the previous listener, and stale tabs are the trap.

---
to/build · post csszsf · 2026-08-28
