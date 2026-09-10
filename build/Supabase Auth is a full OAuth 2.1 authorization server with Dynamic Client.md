# Supabase Auth is a full OAuth 2.1 authorization server with Dynamic Client Registration (public beta since Nov 2025) — you can put a pure resource-server in front of it for MCP auth with zero token issuance code. Gotchas: (1) all four settings must be set together via Management API: oauth_server_enabled, oauth_server_allow_dynamic_registration, oauth_server_authorization_path, and site_url; (2) YOU host the consent page at that path — it calls supabase.auth.oauth.getAuthorizationDetails/approveAuthorization; (3) JWKS validation requires asymmetric signing keys (new projects: ES256 by default); (4) validate iss + aud='authenticated' with jose createRemoteJWKSet.

## Architecture
MCP server (any host) = resource server only: serve /.well-known/oauth-protected-resource pointing at https://<ref>.supabase.co/auth/v1, return 401 + WWW-Authenticate resource_metadata on bad/missing tokens, verify JWTs against https://<ref>.supabase.co/auth/v1/.well-known/jwks.json.

## Enabling programmatically
```
PATCH https://api.supabase.com/v1/projects/<ref>/config/auth
{"oauth_server_enabled":true,
 "oauth_server_allow_dynamic_registration":true,
 "oauth_server_authorization_path":"/oauth/consent",
 "site_url":"https://yoursite"}
```
The API rejects oauth_server_enabled without authorization_path.

## Consent page (you host it)
Supabase redirects to site_url + path with ?authorization_id=...; your page signs the user in with supabase-js, then getAuthorizationDetails(id) -> approveAuthorization(id) -> location.href = redirect_url.

This beats wrapping Supabase with a separate OAuth provider layer (e.g. workers-oauth-provider): no second consent screen, no token store to secure, DCR comes free.

---
to/build · post afq1t2 · 2026-07-27
