---
tags:

- mcp
- oauth2
- authentication
- security
- web-development
- advanced

---

# MCP Authorization: How a Remote MCP Server Authenticates Its Callers

A remote [MCP](../machine-learning/fastmcp-knowledge-base-server.md) server is an
HTTP endpoint that an LLM client (Claude, an IDE agent) calls on a user's behalf.
Anyone who can reach the URL can send requests, so the server needs to know **who is
calling** and **what they approved**. MCP does not invent a new scheme for this: it
reuses [OAuth2](oauth2-jwt-authentication.md) and a handful of small RFCs whose only
job is to let a client that has *never heard of this server* work out how to log in
to it, with nothing but the URL.

That last part is the whole point. The user pastes one link; everything else (where
to log in, how to register, how to get a token) is discovered by the client.

## The Roles

- **MCP server** = the OAuth **resource server**. It holds the tools and data, and
  accepts requests carrying `Authorization: Bearer <access token>`.
- **Authorization server (AS)** = issues those tokens after the user approves. It can
  be the MCP server itself or a separate identity provider.
- **MCP client** = the OAuth **client**. For a hosted assistant it is a *public*
  client: it cannot keep a secret, so it relies on PKCE instead.
- **User** = the **resource owner**, the one whose data the tools touch.

Every token the server accepts must resolve to exactly one user, and every tool must
act as that user only. The token is the identity; tools never take a "user id"
argument.

## The Handshake, Step by Step

```text
Client                         MCP server / AS                    User (browser)
  | POST /mcp (no token) ---------> |
  | <-- 401 + WWW-Authenticate:     |
  |     resource_metadata=<url>     |
  | GET <resource_metadata> ------> |  who is the AS for this resource?
  | GET /.well-known/oauth-authorization-server ->  endpoints + capabilities
  | POST /register ---------------> |  Dynamic Client Registration -> client_id
  | open /authorize?...&code_challenge=... ------------------------> login + consent
  | <-- redirect to client callback with ?code=... ----------------- approve
  | POST /token (code + code_verifier) -> access token + refresh token
  | POST /mcp  Authorization: Bearer ...   -> tools work
```

Each arrow is a named standard:

1. **The 401 challenge** (RFC 9110 + RFC 9728). An unauthenticated request gets
   `401` with `WWW-Authenticate: Bearer resource_metadata="https://host/.well-known/oauth-protected-resource"`.
   This header is the client's only clue, so it has to be right.
2. **Protected Resource Metadata** (RFC 9728). That document says which
   authorization server(s) protect this resource:
   `{"resource": ".../mcp", "authorization_servers": ["https://host"]}`.
3. **Authorization Server Metadata** (RFC 8414). The AS publishes its
   `authorization_endpoint`, `token_endpoint`, `registration_endpoint`, supported
   PKCE methods, and so on, at a well-known path.
4. **Dynamic Client Registration, DCR** (RFC 7591). The client POSTs its name and
   redirect URIs and gets back a fresh `client_id`. This is what removes the
   "create an app and paste a client id" step. Without DCR, the user has to register
   the client by hand and type the id into the assistant's settings.
5. **Authorization code + PKCE** (OAuth 2.1). The client sends the user to
   `/authorize` with a `code_challenge`; the AS authenticates the user, asks for
   consent, and redirects back with a one-time `code`. The client redeems it at
   `/token` together with the `code_verifier`; the AS checks
   `SHA256(verifier) == challenge`. A stolen code is useless without the verifier.
6. **Bearer token on every call.** Access tokens are short-lived; the refresh token
   gets a new one without bothering the user again.

!!! note
    The spec also asks clients to send the target in a `resource` parameter
    (RFC 8707 resource indicators) so a token is minted for *this* MCP server only
    and cannot be replayed against another. A server that issues its own tokens only
    for itself gets that property for free; one that delegates to a shared identity
    provider has to check the audience carefully.

## Two Ways to Provide the Authorization Server

### 1. The MCP server is its own authorization server

The server implements `/register`, `/authorize`, `/token` and the two metadata
documents itself, and keeps clients, codes and tokens in its own store.

- **Pros:** works with any client (DCR is available), tokens are bound to this server
  only, and the consent screen can say exactly what is being granted.
- **Cons:** a small security-critical component to own and test: PKCE checking,
  single-use codes, redirect-URI validation, token hashing, rotation.

The "login" step does not need passwords: `/authorize` can just send the user to the
app's own page, where they are already signed in (through whatever SSO the app uses),
and show an **Approve / Deny** screen. The approval records a grant tied to that
user's stable id.

### 2. Delegate to an external identity provider

The MCP server only validates tokens issued by a provider such as
[Cognito](../devops-tools/aws/cognito.md), and points `authorization_servers` at it.

- **Pros:** no token issuance code to maintain; one place for users and MFA.
- **Cons:** the provider must support what the client expects. Cognito, notably, has
  **no Dynamic Client Registration**, so a connector can only work if someone
  pre-registers a client and gives the assistant that client id by hand (and a shim in
  front of the provider usually has to paper over the missing discovery documents).
  Tokens are also issued for the whole pool, so the server must check the client and
  audience to know a token was meant for it.

!!! tip "Rule of thumb"
    "Paste a link and it works, for several different assistant accounts" means the
    server must speak DCR, which in practice means it runs its own small authorization
    server. Delegating to a provider is simpler when the clients are known in advance.

## Design Decisions That Matter

**Public clients only.** `token_endpoint_auth_method: none`. There is no client
secret to leak; PKCE binds each code to the client instance that started the flow.

**Exact redirect-URI matching, and validate it before anything else.** An error in
`client_id` or `redirect_uri` must be shown to the user, never redirected to the
supplied URI, otherwise `/authorize` becomes an open redirector. Only after those two
pass is it safe to bounce other errors back to the client with `?error=...&state=...`.

**Restrict who may register.** DCR is unauthenticated by design, so anyone can
register. Limit it: accept only the assistant vendor's callback hosts, rate-limit
registrations, and keep client records small and expiring. A registered client alone
grants nothing; the user's consent is what creates a grant.

**Consent is bound to a user, not a client.** A grant records *which user approved
which client*. The access token resolves to that user's id, so a token approved by
user A can never read user B's data, even though many assistant accounts share the
same registered client or callback URL.

**Hash what you store.** Authorization codes, access tokens and refresh tokens are
stored as digests (SHA-256 is enough for high-entropy random values). A database
leak then yields nothing usable.

**Short access tokens, rotating refresh tokens.** Access tokens live about an hour.
Each refresh returns a *new* refresh token and invalidates the old one. If an old
refresh token is ever presented again, treat it as theft and **revoke the whole
grant**, forcing a fresh consent.

**Make grants visible and revocable.** A "Connections" page listing each connected
assistant, with a Disconnect button, is the user's kill switch. Deleting the grant
invalidates its refresh and access tokens at once.

**Single-use, short-lived codes and pending requests.** An authorization code lives
for about a minute and can be redeemed once; a pending consent request expires in
minutes. Table TTLs make cleanup automatic.

## Multiple Assistant Accounts, One User

Several people, or several assistant accounts of the same person, can connect to the
same MCP server. Each connection does its own DCR + consent and ends up with its own
grant row, all pointing at the approving user's id. That gives:

- per-connection revocation (disconnect one without touching the others);
- no cross-talk, because the user id comes from the grant, not from anything the
  assistant sends;
- an audit trail of "which client was approved, when".

## Gotchas Seen in Practice

- **The 401 must carry the right `WWW-Authenticate`.** Frameworks and proxies often
  emit their own; the client then never finds the metadata and reports a generic
  "couldn't connect".
- **Discovery is path-sensitive.** Clients try both
  `/.well-known/oauth-protected-resource` and the path-suffixed
  `/.well-known/oauth-protected-resource/mcp`. Serve both.
- **A CDN in front can break it quietly.** The well-known documents, `/mcp`,
  `/oauth/*` and the consent API must all be routed to the backend and must not be
  cached (`Cache-Control: no-store` on token responses).
- **Stateless transport.** A stateless Streamable-HTTP server only routes `POST`; an
  unauthenticated `GET` then returns `405` instead of the `401` that starts the OAuth
  flow, so the auth check has to run before routing.
- **Tool limits follow from where the server runs.** A hosted server has no local
  files or browser, so tools that need them (render a PNG, upload a local image) stay
  in a local stdio server.
- **Client-side state is sticky.** After the server's auth model changes, the
  assistant must remove and re-add the connector; stale registrations point at
  endpoints that no longer exist.

## Summary

- MCP auth is plain OAuth2 with **discovery** glued on: 401 → resource metadata (RFC
  9728) → AS metadata (RFC 8414) → dynamic registration (RFC 7591) → code + PKCE →
  bearer token.
- The MCP server is a resource server; whether it also acts as the authorization
  server decides whether "just a link" onboarding is possible.
- No DCR (as in Cognito) means manual client registration; running a small
  authorization server with DCR removes it.
- Bind every grant to the approving user's id, hash all secrets, rotate refresh
  tokens and revoke the grant on reuse, restrict DCR, and give users a revoke page.
- Validate `client_id` and `redirect_uri` before ever redirecting.

## Related Articles

- [OAuth2 & JWT Authentication](oauth2-jwt-authentication.md) — the grant types,
  PKCE, token validation and refresh-rotation basics this builds on.
- [Single Sign-On with OIDC](single-sign-on-oidc.md) — how the user's login behind
  the consent screen works when several apps share one identity provider.
- [FastMCP: Building and Deploying an MCP Server](../machine-learning/fastmcp-knowledge-base-server.md)
  — the server side: tools, transports, deployment.
- [AWS Cognito](../devops-tools/aws/cognito.md) — the identity provider used in the
  delegated variant, and its missing DCR.
