# SearchPoint Integration Development

Build your own MCP client or tool integration against **SearchPoint Desktop**.

SearchPoint runs a local [Model Context Protocol](https://modelcontextprotocol.io) server that lets an AI client search the user's documents, read file metadata, manage favorites, tags, and local folders, and query facts read from their documents.

> The server runs **inside the SearchPoint Desktop app**, on loopback only. It is available while the app is running and the user is signed in. Users turn it on in **Settings > MCP**, where they can also connect Cursor and review connected clients.

---

## Quick start

| | |
| --- | --- |
| Endpoint | `http://127.0.0.1:8742/mcp` |
| Transport | Streamable HTTP carrying JSON-RPC 2.0, one endpoint, `POST` |
| Auth | OAuth 2.1, discovery + Dynamic Client Registration + PKCE (`S256`) |
| Protocol versions | `2025-06-18` (latest), `2025-03-26`, `2024-11-05` |

Discovery documents:

```
http://127.0.0.1:8742/.well-known/oauth-protected-resource
http://127.0.0.1:8742/.well-known/oauth-authorization-server
```

A request without a valid Bearer token gets `401` with a challenge header pointing at the resource metadata:

```
WWW-Authenticate: Bearer resource_metadata="http://127.0.0.1:8742/.well-known/oauth-protected-resource"
```

`GET /mcp` returns `405`. Requests carrying an `Origin` header are rejected with `403`, so browser pages cannot reach the server.

The whole OAuth flow, including refresh rotation and revocation, is documented in [docs/authentication.md](docs/authentication.md). The tool reference is in [docs/tools.md](docs/tools.md).

---

## Prompt for your agent

Paste this into the agent that will write the integration:

```text
Add SearchPoint as an MCP server to my tool.

Transport: Streamable HTTP (JSON-RPC 2.0), one endpoint:
  http://127.0.0.1:8742/mcp

Auth: OAuth 2.1. Discover the server from:
  http://127.0.0.1:8742/.well-known/oauth-protected-resource
  http://127.0.0.1:8742/.well-known/oauth-authorization-server
Register with dynamic client registration (POST /register), then run the
authorization code flow with PKCE (S256) and a loopback redirect URI
(http://127.0.0.1:<port>/callback). Never send owner_email — the owner is
implicit.

Protocol: negotiate one of 2025-06-18 (latest), 2025-03-26, 2024-11-05 in
`initialize`.

Then call the tools listed in docs/tools.md. `search_file` also accepts
the Advanced search groups: `all` (AND), `any` (OR), and `none` (NOT).
Store the tokens, refresh them, and handle JSON-RPC errors.
```

A working client needs to:

1. Send `initialize`, then `notifications/initialized`.
2. Discover the authorization server from the `401` challenge.
3. Register itself with `POST /register`.
4. Open `/authorize` with `code_challenge_method=S256` and a loopback redirect URI, then wait for the user to approve the client in the SearchPoint app.
5. Exchange the code at `POST /token`, refresh before the access token expires, and send `Authorization: Bearer <token>` on every `POST /mcp`.
6. Handle `tools/list` and `tools/call`.

---

## Tools

The desktop server exposes 17 tools. The web client exposes only `search_file` and `get_file`; every other tool is desktop only.

| Tool | Purpose |
| --- | --- |
| `search_file` | Full-text search, with the Advanced search groups (`all`/`any`/`none`) |
| `get_file` | Full metadata for one document |
| `add_local_source` | Register and index a local folder |
| `remove_local_source` | Unregister a folder and delete its index |
| `list_local_sources` | List active local folders |
| `add_favorite` | Favorite a file |
| `remove_favorite` | Unfavorite a file |
| `list_favorites` | List favorites, newest first |
| `list_file_tags` | List the tags on one file |
| `list_tags` | List every tag with its file count |
| `list_files_by_tag` | List the files carrying one tag |
| `append_tag` | Add one or more tags to a file |
| `remove_tag` | Remove one or more tags from a file |
| `get_document_facts_guide` | How to ask what documents say. Read it first |
| `search_document_facts` | Find documents by what they say: type, parties, dates, amounts |
| `get_document_facts` | Every fact about one document, organization or person, with its quote |
| `query_document_facts` | Read-only SPARQL over the document facts |

Parameters, response shapes, and examples are in [docs/tools.md](docs/tools.md).

---

## Errors

Standard JSON-RPC 2.0 codes:

| Code | Meaning |
| --- | --- |
| `-32700` | Parse error |
| `-32600` | Invalid request |
| `-32601` | Method not found / unknown tool |
| `-32602` | Invalid params (wrong type, missing required, unknown key, relative path) |
| `-32603` | Internal error |

A tool failure comes back as a normal JSON-RPC result with `isError: true` and a readable message, not a protocol error. Notifications (messages without an `id`) get `202` with no body. Batches are rejected.

---

## Good to know

- **One user at a time.** The server acts for whoever is signed in to the desktop app. Never send `owner_email`; the owner is implicit. Connections belong to the computer, so signing in as another account keeps the connection and changes whose data tool calls act on.
- **The user must approve you.** The first `/authorize` for a new client shows a consent dialog in the app. An unknown or inactive client cannot call tools.
- **New apps start read-only.** A newly connected client can call only `search_file` and `get_file`. The user enables the rest per app in **Settings > MCP > Connected Apps > Tools**. A disabled tool still appears in `tools/list` with a note, and calling it returns `isError: true`.
- **Keep it local.** Redirect URIs must be loopback (`http://127.0.0.1:<port>/...` or `localhost`) with no fragment or userinfo. `https://www.cursor.com/agents/mcp/oauth/callback` is the only non-loopback exception, reserved for Cursor.
- **No static secrets.** Dynamic client registration issues a public `client_id` and no secret. Tokens are opaque bearer credentials.
- **The server can disappear.** If the app quits or the user signs out, connections fail. Signing out makes every OAuth endpoint answer `401` without stopping the listener.
- **Document facts need Smart search.** `search_document_facts`, `get_document_facts`, and `query_document_facts` return an error explaining why when Smart search is off, no one is signed in, or a file has not been read yet. `search_file` keeps working.

---

## License

See the SearchPoint product terms. This repository contains documentation only.
