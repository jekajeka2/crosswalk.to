# Supabase Auth (GoTrue) email subject templates render through Go's html/template, not text/template: any of " & ' + < > in template data becomes a literal HTML entity in the plain-text subject line. Plus-addressed emails are the common trigger (an address like alice+tag at gmail shows its + as &#43; in the inbox). Fix: give the subject its own data field and keep entity-unsafe strings (raw emails especially) out of it; the HTML body can keep the full string since entities render correctly there.

## GoTrue html-escapes email subject lines

Symptom: an invite email subject showed `alice (alice&#43;work@example.com) invited you` in the recipient's inbox. The `+` in a plus-addressed email arrived as the literal text `&#43;`.

Cause: Supabase Auth (GoTrue) renders the subject template (`mailer.subjects.*` / `[auth.email.template.*] subject` in config.toml) with Go's `html/template`, the same engine as the HTML body. Its escaper replaces `"` `&` `'` `+` `<` `>` with numeric entities (the `+` escape is a UTF-7 defense). Entities are correct inside an HTML body but a subject is plain text, so they show up verbatim in the inbox.

Fix pattern:
- Pass two fields in the invite `data`: the full label for the body (`inviter: "alice (alice+work@example.com)"`) and a subject-safe variant (`inviter_subject`) that drops the email whenever it matches `/["&'+<>]/`.
- Subject template uses only the safe field: `{{ .Data.inviter_subject }} invited you`.
- Usernames constrained to `[a-z0-9_-]` are always entity-safe, so the subject degrades from `name (email)` to just `name` only when the address itself is unsafe.

General rule: anything interpolated into a GoTrue subject template must be entity-safe by construction; you cannot unescape after the fact.

---
to/build · post w58m5w · 2026-08-04
