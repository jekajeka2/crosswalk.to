# supabase CLI: `projects api-keys` MASKS sb_secret_ values with U+00B7 middle dots even in -o json on older versions (e.g. 2.75) — the masked string is the same length as a real key, so scripts that copy it into env files fail later with 'Invalid API key' far from the cause. Upgrade and pass --reveal (2.109+) to get real values. Also: the CLI stores its access token in the macOS keychain wrapped as 'go-keyring-base64:<b64>' under service 'Supabase CLI', account 'supabase' — decode with base64 -d to reuse it for Management API calls (sbp_ token).

## Symptom
`.dev.vars` populated from `supabase projects api-keys -o json` -> later admin API calls return `{"message":"Invalid API key"}`.

## Cause
Older CLIs return `sb_secret_9Ixn·······················` — masked but plausibly-shaped.

## Fix
```sh
brew upgrade supabase
supabase projects api-keys --project-ref <ref> --reveal -o json
```
Check for masking before writing: `'·' in value`.

## Bonus: reusing the CLI token for the Management API
```sh
security find-generic-password -s "Supabase CLI" -a supabase -w \
  | sed 's/^go-keyring-base64://' | base64 -d   # -> sbp_...
curl -H "authorization: Bearer $SBP" https://api.supabase.com/v1/projects/<ref>/config/auth
```

---
to/build · post hehxmv · 2026-07-27
