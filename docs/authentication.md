# Authentication

The SearchPoint MCP server is its own OAuth 2.1 authorization server and protected resource server, bound to loopback. Clients register themselves with Dynamic Client Registration (DCR), ask the user for approval inside the SearchPoint app, and exchange the authorization code for tokens. No client ships a secret and no client needs a pre-shared client ID.

## Constants

| Name | Value |
| --- | --- |
| Issuer | `http://127.0.0.1:8742` |
| Resource | `http://127.0.0.1:8742/mcp` |
| Also served as | `http://localhost:8742` |

The server echoes back whichever host the client used (`127.0.0.1` or `localhost`) in the metadata documents and in the challenge header, so both spellings work end to end.

## Discovery

A client that receives `401` on `POST /mcp` reads:

```
WWW-Authenticate: Bearer resource_metadata="http://127.0.0.1:8742/.well-known/oauth-protected-resource"
```

`GET /.well-known/oauth-protected-resource`

```json
{
  "resource": "http://127.0.0.1:8742/mcp",
  "authorization_servers": ["http://127.0.0.1:8742"]
}
```

`GET /.well-known/oauth-authorization-server`

```json
{
  "issuer": "http://127.0.0.1:8742",
  "authorization_endpoint": "http://127.0.0.1:8742/authorize",
  "token_endpoint": "http://127.0.0.1:8742/token",
  "registration_endpoint": "http://127.0.0.1:8742/register",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"]
}
```

Only `code` responses and `S256` PKCE are advertised, and both are enforced server side.

## Registration (DCR)

`POST /register` takes a `ClientRegistration`:

```
client_id      optional
client_name    optional
redirect_uris  required, one or more
```

If `client_id` is absent, the server generates one. A caller-supplied `client_id` is rejected, except for Cursor's known public client metadata. A successful registration returns `201 Created`:

```json
{
  "client_id": "...",
  "client_name": "...",
  "redirect_uris": ["http://127.0.0.1:54321/callback"]
}
```

Invalid metadata returns `400 invalid_client_metadata`.

Redirect URI rules:

- Scheme must be `http` (except Cursor's callback).
- Host must be `localhost` or a loopback IP.
- No fragment, no userinfo, no password.
- `https://www.cursor.com/agents/mcp/oauth/callback` is the single non-loopback exception.

`client_name` defaults to `MCP client` when omitted. Metadata for a client that already holds an active grant is never overwritten, so re-registering cannot silently swap a trusted client's identity.

Registration alone grants nothing. The client must still complete `/authorize` and be approved by the user.

## Client ID Metadata Documents

Some clients do not call `/register`. They present an `https://` URL as their `client_id` instead. The server fetches that URL, with redirects disabled and a 10 second timeout, and reads the metadata document from it. The URL must be `https://` and carry no fragment.

This path ends in the same consent flow as DCR, so a metadata-document client never skips user approval.

## Authorization code flow

`GET /authorize` runs these steps:

1. `response_type` must be `code`.
2. The client is resolved from DCR or its metadata document. Unknown means `invalid_client`.
3. `redirect_uri` must match a registered URI exactly. A mismatch is `invalid_request`, and no redirect happens on failure.
4. `code_challenge` must be 43 to 128 characters from `[A-Za-z0-9-._~]`, and `code_challenge_method` must be `S256`.
5. If the client already has an active grant, a code is issued immediately and the consent prompt is skipped.
6. Otherwise the app shows a consent dialog and waits up to 5 minutes.
7. Denial or timeout redirects with `error=access_denied`.
8. On approval the grant is created and the code is issued.

Codes are 32 random bytes encoded `URL_SAFE_NO_PAD`. Only the SHA-256 hash is stored, and a code expires after 5 minutes.

## PKCE

The server never generates or stores a verifier. It stores `code_challenge` with the code and recomputes it during exchange:

```
challenge = BASE64URL(SHA256(code_verifier))
```

## Token endpoint

`POST /token` accepts `application/x-www-form-urlencoded`:

- `grant_type=authorization_code` with `client_id`, `redirect_uri`, `code`, `code_verifier`, all required.
- `grant_type=refresh_token` with `refresh_token`.
- Anything else returns `400 unsupported_grant_type`.

Success:

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "token_type": "Bearer",
  "expires_in": 900
}
```

Responses carry `Cache-Control: no-store` and `Pragma: no-cache`. Failures return `400 invalid_grant`.

### Code exchange

The exchange runs in one transaction:

1. Validate the verifier shape.
2. Load the code with its grant, requiring an unexpired code.
3. Reject on client mismatch, redirect mismatch, challenge mismatch, an already-used code, expiry, or a grant that is not active.
4. Mark the code used atomically, requiring exactly one affected row, so two concurrent redemptions cannot both win.
5. Issue the token pair and commit.

### Lifetimes

| Token | Lifetime |
| --- | --- |
| Access | 15 minutes |
| Refresh | 30 days |

Tokens are 32 random bytes encoded `URL_SAFE_NO_PAD`. Only SHA-256 hashes reach storage. Plaintext exists only in memory and in HTTP responses.

### Refresh with rotation

Every refresh rotates the pair. Refresh tokens carry a `family_id` shared by every rotation of the same grant.

If a refresh token is presented after it was already rotated while the grant is still active, the server treats it as a replay: the whole grant and all of its access and refresh tokens are revoked. Otherwise the old token is marked rotated and a new pair is issued under the same family.

## Using a token

Send `Authorization: Bearer <access_token>` on every `POST /mcp`. The server requires an unexpired, unrevoked token whose grant is still active, and updates `last_used_at` for the client and the grant.

## Revocation

The user can revoke a connected client from **Settings > MCP > Connected Apps**. Revoking flips the grant to `revoked` and revokes every access and refresh token in that grant in one transaction. The client registration survives, so a later reconnect reuses the same client identity but still needs approval.

## Signed out

Without a signed-in user, every OAuth endpoint answers `401 invalid_request`. Signing out is enough to shut off MCP access without stopping the listener.

## Error codes

| Error | Where |
| --- | --- |
| `invalid_client` | Unknown client at `/authorize` |
| `invalid_request` | Bad request shape, redirect mismatch, signed out |
| `invalid_client_metadata` | Rejected `/register` |
| `invalid_grant` | Bad, expired, used, or replayed code or token |
| `access_denied` | User denied consent, or consent timed out |
| `unsupported_grant_type` | Grant type other than `authorization_code` or `refresh_token` |
