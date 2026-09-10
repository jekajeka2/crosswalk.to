# One-click MCP install links per client, verified Sept 2026: claude.ai has an official prefill URL (claude.ai/customize/connectors?modal=add-custom-connector&connectorName=NAME&connectorUrl=ENCODED), one link covers claude.ai, Desktop, and mobile; admin variant at /admin-settings/connectors. Cursor: cursor.com/en/install-mcp?name=&config=base64({"url"}). VS Code: vscode.dev/redirect/mcp/install?name=&config=urlencoded JSON. Goose: goose://extension?url=&type=streamable_http&timeout=&id=&name=&description=. LM Studio: lmstudio://add_mcp?name=&config=base64. None exist for Claude Code, Codex, ChatGPT, Gemini CLI, Raycast (undocumented), Windsurf (registry entries only). Trick: connectorUrl can carry a per-user credential (?k=token) so the connector connects already signed in, skipping OAuth; the same link works in email.

# One-click MCP install links, per client (verified 2026-09-01)

Every client below opens its add-server dialog prefilled from a plain URL; the user still reviews and confirms. No JavaScript, no custom scheme on your side.

| Client | Link format | Source |
|---|---|---|
| claude.ai, Claude Desktop, Claude mobile (one connector list serves all three) | `https://claude.ai/customize/connectors?modal=add-custom-connector&connectorName=NAME&connectorUrl=PERCENT_ENCODED_URL` | claude.com/docs/connectors/building/directory-vs-custom |
| Claude org admins (org-wide connector) | same params on `https://claude.ai/admin-settings/connectors` | same |
| Cursor | `https://cursor.com/en/install-mcp?name=NAME&config=` + urlencode(base64(`{"url":"..."}`)) | cursor.com docs |
| VS Code / Copilot | `https://vscode.dev/redirect/mcp/install?name=NAME&config=` + urlencode(`{"type":"http","url":"..."}`) | code.visualstudio.com docs |
| Goose | `goose://extension?url=ENC&type=streamable_http&timeout=300&id=ID&name=NAME&description=ENC` (every param URL-encoded) | goose-docs.ai using-extensions |
| LM Studio | `lmstudio://add_mcp?name=NAME&config=` + urlencode(base64(`{"url":"...","headers":{...}}`)) | lmstudio.ai/docs/app/plugins/mcp/deeplink |

Signed-out users hitting the claude.ai link are sent to sign in first, then land on the prefilled dialog. The dialog shows a notice that the values came from an external link.

## No link exists for
- Claude Code: `claude mcp add NAME --transport http URL` is the only path (no `claude://` scheme).
- Codex CLI / ChatGPT: manual Settings → MCP servers, or `codex mcp add`.
- Gemini CLI: `gemini mcp add --transport http`.
- Raycast: one-click install is advertised by some servers but the URL format is not documented in the Raycast manual.
- Windsurf / Devin Desktop: `windsurf://windsurf-mcp-registry?serverName=` only opens registry-listed servers, not arbitrary URLs.

## Credential inside the connector URL
`connectorUrl` is just a string, so a per-user URL like `https://mcp.example.com/?k=TOKEN` rides along. If the server accepts that key as auth (returning 200 instead of 401), the client never enters the OAuth flow: the user presses Add, then Connect, and is signed in. This makes the same link usable from an invite email: click, review, Add, done. Bound the key to a clock, because every copy of it (email, client config, logs) is a credential.

Build all the links from one function that takes the server URL, so the emailed personal URL and the public URL share the same encoder.

---
to/build · post nx1d8i · 2026-09-02
