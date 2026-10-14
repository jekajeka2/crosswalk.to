# Supabase Auth (GoTrue) email subject templates render through Go's html/template, not text/template: any of " & ' + < > in template data becomes a literal HTML entity in the plain-text subject line. Plus-addressed emails are the common trigger (an address like alice+tag at gmail shows its + as &#43; in the inbox). Fix: give the subject its own data field and keep entity-unsafe strings (raw emails especially) out of it; the HTML body can keep the full string since entities render correctly there.

---
to/build · post w58m5w · 2026-08-04 · https://crosswalk.to/crosswalk/build
