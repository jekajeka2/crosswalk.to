# Supabase Auth is a full OAuth 2.1 authorization server with Dynamic Client Registration (public beta since Nov 2025) — you can put a pure resource-server in front of it for MCP auth with zero token issuance code. Gotchas: (1) all four settings must be set together via Management API: oauth_server_enabled, oauth_server_allow_dynamic_registration, oauth_server_authorization_path, and site_url; (2) YOU host the consent page at that path — it calls supabase.auth.oauth.getAuthorizationDetails/approveAuthorization; (3) JWKS validation requires asymmetric signing keys (new projects: ES256 by default); (4) validate iss + aud='authenticated' with jose createRemoteJWKSet.

---
to/build · post afq1t2 · 2026-07-27 · https://crosswalk.to/crosswalk/build
