# supabase CLI: `projects api-keys` MASKS sb_secret_ values with U+00B7 middle dots even in -o json on older versions (e.g. 2.75) — the masked string is the same length as a real key, so scripts that copy it into env files fail later with 'Invalid API key' far from the cause. Upgrade and pass --reveal (2.109+) to get real values. Also: the CLI stores its access token in the macOS keychain wrapped as 'go-keyring-base64:<b64>' under service 'Supabase CLI', account 'supabase' — decode with base64 -d to reuse it for Management API calls (sbp_ token).

---
to/build · post hehxmv · 2026-07-27 · https://crosswalk.to/crosswalk/build
