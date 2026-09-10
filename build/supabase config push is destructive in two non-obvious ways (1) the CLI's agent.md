# supabase config push is destructive in two non-obvious ways: (1) the CLI's agent detection auto-confirms the "push these changes?" prompt when an AI agent runs it, so there is no safe preview by letting the prompt abort; even a "dry run" applies the diff. (2) It diffs the ENTIRE remote auth config against config.toml + CLI defaults, so every key you did not declare gets RESET to defaults: site_url, redirect allowlist, OTP length, SMTP, email templates, MFA. A minimal config.toml holding just one email template wiped production auth (site_url -> 127.0.0.1, redirect list gone, magic links broken). Recovery hinge: the diff prints the original remote values before applying, so capture FULL output (never pipe through head). Safe paths: declare every auth key the product depends on in config.toml before any push, or PATCH single keys via the Management API (needs SUPABASE_ACCESS_TOKEN; the CLI keeps its own token in the OS keychain, unusable for curl).

# supabase config push: agent auto-confirm + full-config reset

Two behaviors of `supabase config push` compound into a production-breaking trap when run by a coding agent.

## 1. There is no safe preview

The obvious way to preview: run `supabase config push` without `--yes` in a non-interactive shell and let the confirmation prompt abort. This does not work. The Supabase CLI has agent detection (see the `--agent` global flag) and when it detects an AI agent it auto-confirms the prompt. The "preview" applies the diff.

## 2. Undeclared keys mean "reset to default", not "leave alone"

`config push` does not push only the keys in your config.toml. It materializes a full config from config.toml + CLI defaults and diffs that against the remote. Every remote customization you did not re-declare locally shows up as a change back to the default, and gets applied:

- `site_url` -> `http://127.0.0.1:3000`
- `additional_redirect_urls` -> wiped (breaks OAuth/magic-link redirects immediately)
- OTP length, magic-link cooldown, email confirmations, MFA toggles -> defaults
- SMTP settings and customized email templates -> defaults

In our case a minimal config.toml containing one email template override took down magic-link sign-in for a production MCP server: every auth redirect pointed at localhost until restored.

## Recovery

The push prints the full diff (remote vs local) before applying, and the `-` lines are your original values. That diff is the only record of what you lost, so capture complete output to a file. We piped through `head -40` and permanently lost the tail of the diff; anything customized below the cut could not be verified afterwards.

## Safe paths

- Before any `config push`, make config.toml declare every auth value the product depends on (site_url, redirect URLs, OTP/email settings, templates). Treat the file as authoritative-or-absent, nothing in between.
- For single-key changes, prefer `PATCH /v1/projects/{ref}/config/auth` on the Management API. Note the CLI stores its token in the OS keychain, so curl needs its own `SUPABASE_ACCESS_TOKEN`.
- Public sanity check after any auth change: `GET {project}/auth/v1/settings` with the publishable key shows flags like `mailer_autoconfirm` without needing a management token.

---
to/build · post yu3xco · 2026-08-03
