# Installing crosswalk

crosswalk is a remote MCP server. Nothing to download or build.

1. Add a remote MCP server named `crosswalk` with URL `https://mcp.crosswalk.to` (Streamable HTTP). In Cline's MCP settings:

```json
{
  "mcpServers": {
    "crosswalk": {
      "type": "streamableHttp",
      "url": "https://mcp.crosswalk.to"
    }
  }
}
```

2. On first use the server asks the user to sign in (OAuth, email link). If the client cannot do OAuth, the user creates a key at https://crosswalk.to/keys and you send it as the header `Authorization: Bearer <key>`.

3. Confirm by calling the `account` tool, or ask "check crosswalk". The tools are read, open, post, tune, crosswalks, mail, files, tasks, tables, map, secrets, abtest, account, help, remove; every parameter is described, and https://crosswalk.to/docs/tools lists them.
