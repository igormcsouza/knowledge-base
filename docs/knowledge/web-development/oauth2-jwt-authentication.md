---
tags:

- web-development
- oauth2
- jwt
- authentication
- security

---

# OAuth2 & JWT Authentication

OAuth2 and JWTs get used together so often that they're easy to conflate,
but they answer different questions: OAuth2 is about **delegated
authorization** ("can this app act on my behalf, with these specific
permissions"), and a JWT is just a **token format** — a portable, signed
claim that happens to be a convenient way to carry the result of an OAuth2 (or
any other) authentication/authorization decision. Understanding where the
line is between them makes both far less confusing.

## OAuth2 Is Authorization, Not Authentication

OAuth2's actual job: let a **client** (a third-party app) get limited access
to a **resource owner's** (the user's) data on a **resource server**, without
the client ever seeing the user's password, and scoped to exactly the
permissions the user consented to. That's authorization — "what can this
client do." By itself, OAuth2 says nothing about "who is this user" — that's
what [OIDC](#oidc-authentication-on-top-of-oauth2) adds on top.

### Grant Types

- **Authorization code** — the default for anything involving a browser. The
  user is redirected to the authorization server, logs in there (the client
  never sees credentials), and the server redirects back with a short-lived
  `code`, which the client's *backend* exchanges for tokens directly with the
  authorization server (a server-to-server call, not visible to the browser).
  Adding **PKCE** (Proof Key for Code Exchange) makes this safe for public
  clients too (SPAs, mobile apps) that can't hold a client secret.
- **Client credentials** — for service-to-service calls with no user
  involved at all: a service authenticates with its own client ID/secret and
  gets a token representing itself, not a user.

  ```http
  POST /oauth/token
  Content-Type: application/x-www-form-urlencoded

  grant_type=client_credentials&client_id=svc-a&client_secret=***&scope=orders:read
  ```

- **Implicit grant** — returned tokens directly in the browser redirect URL
  fragment, skipping the code exchange step. **Deprecated** by the OAuth2
  security best-practices spec: tokens end up in browser history, referrer
  headers, and JS-accessible URL state, with no way to keep a client secret
  out of it. Use authorization code + PKCE instead, even for pure SPAs.
- **Password grant (Resource Owner Password Credentials)** — the client
  collects the user's username/password directly and trades them for a
  token. **Discouraged**: it defeats OAuth2's entire premise of the client
  never seeing the user's credentials, and trains users to type passwords
  into arbitrary third-party UIs.

!!! tip "Pro Tip"
    If you're starting a new integration today: browser-involved flow →
    authorization code + PKCE. Machine-to-machine, no user → client
    credentials. If you find yourself reaching for implicit or password
    grants, that's almost always a sign the library/guide you're following is
    outdated.

## OIDC: Authentication on Top of OAuth2

**OpenID Connect (OIDC)** is a thin, standardized identity layer built on
top of OAuth2's authorization flows. It adds exactly what plain OAuth2
doesn't guarantee: a standardized **ID token** (specifically a JWT) asserting
who the user is, plus a standard `/userinfo` endpoint and a discovery
document (`/.well-known/openid-configuration`) so clients don't need
provider-specific integration code. In practice, "log in with Google/GitHub/
Cognito" is OIDC — OAuth2's authorization-code flow, plus the identity
assertion OIDC adds on top. [AWS Cognito](../devops-tools/aws/cognito.md) is
a concrete example already covered here: its User Pools issue exactly this
pair — an OIDC ID token for identity, plus an OAuth2 access token scoped for
authorization.

## JWT Structure

A JWT is three base64url-encoded segments separated by dots:
`header.payload.signature`.

```text
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyXzQyIiwiZXhwIjoxNzM5NjAwMDAwfQ.
QRrj9J...signature...
```

```json
// header — decoded
{ "alg": "RS256", "typ": "JWT" }

// payload — decoded
{ "sub": "user_42", "iss": "https://auth.example.com", "aud": "api.example.com",
  "exp": 1739600000, "iat": 1739596400 }
```

!!! warning
    Base64 is **encoding, not encryption** — anyone holding the token can
    decode the payload and read it (paste one into jwt.io to see for
    yourself). Never put a secret, password, or anything sensitive-if-leaked
    into a JWT payload. The signature protects *integrity* (has this been
    tampered with) — it does not provide *confidentiality*.

## Signature Verification

The signature is what makes the claims trustworthy — without verifying it, a
client could hand you any payload it wants and claim it's real.

- **HMAC (HS256)** — a single shared secret both signs and verifies. Simple,
  but every service that needs to *verify* a token also holds the secret
  needed to *mint* one — fine for a single monolith, risky once multiple
  services need to verify tokens, since a compromise of any one of them
  compromises the ability to forge tokens everywhere.
- **RS256 (RSA, asymmetric)** — the issuer signs with a **private** key;
  every consumer verifies with the corresponding **public** key. This is
  exactly why asymmetric signing matters at any real scale: many services
  can safely hold the public key and verify tokens, with zero risk of one of
  them turning around and minting a token, since only the issuer holds the
  private key. This is also what makes JWKS endpoints (like Cognito's) work
  — they publish the public keys openly, and consumers fetch and cache them.

```python
import jwt  # PyJWT

# HS256 — one shared secret, symmetric
decoded = jwt.decode(token, "shared-secret", algorithms=["HS256"])

# RS256 — verify with the issuer's public key only; the private key
# that signed it never needs to leave the auth server
decoded = jwt.decode(token, public_key_pem, algorithms=["RS256"],
                      audience="api.example.com", issuer="https://auth.example.com")
```

## Access Tokens vs. Refresh Tokens

- **Access token** — short-lived (minutes to a couple hours), sent with
  every API request (`Authorization: Bearer <token>`), and what a resource
  server actually verifies to authorize a call.
- **Refresh token** — long-lived, used only against the authorization
  server to obtain a new access token (and often a new refresh token) once
  the current access token expires, without forcing the user to log in
  again.

**Rotation**: each time a refresh token is used, the authorization server
issues a *new* refresh token and invalidates the old one. This turns a
stolen-but-unused refresh token into a detectable event — if both the
legitimate client and an attacker ever try to use the same (now-superseded)
refresh token, the server can recognize the reuse and revoke the entire
token family, rather than a stolen long-lived token silently working forever.

## Common Pitfalls

**Not validating every claim that matters, not just the signature.** A
cryptographically valid signature only proves the token wasn't tampered
with — it says nothing about whether *this* token is the right one for
*this* request unless you also check:

- `exp` (expiry) — reject expired tokens; some libraries don't check this by
  default unless you pass the right option.
- `iss` (issuer) — reject tokens from an issuer you don't trust, so a token
  from a *different* legitimate service using the same JWT library can't be
  replayed against yours.
- `aud` (audience) — reject tokens minted for a *different* API, so a token
  legitimately issued for Service A can't be reused against Service B just
  because both trust the same issuer.

```python
# missing audience/issuer checks — a token meant for a different service
# would pass this if it happens to share the same signing key
jwt.decode(token, public_key_pem, algorithms=["RS256"])  # too permissive

# correct — pins the token to exactly this API and this issuer
jwt.decode(token, public_key_pem, algorithms=["RS256"],
           audience="api.example.com", issuer="https://auth.example.com")
```

**Storage: localStorage vs. httpOnly cookies.** Where the client stashes the
access token trades one attack surface for another:

- **`localStorage`** — readable by any JavaScript running on the page, which
  means an **XSS** vulnerability anywhere in the app can exfiltrate the
  token directly. Not sent automatically, so it needs no separate CSRF
  defense.
- **`httpOnly` cookie** — invisible to JavaScript entirely, so XSS can't
  read the token directly. But the browser attaches cookies automatically
  to matching-origin requests, which reopens **CSRF** — a malicious page can
  trigger a request that carries the cookie, unless it's paired with
  `SameSite` cookie attributes and/or a CSRF token.

Neither storage choice is free of risk — it's XSS-exposure vs.
CSRF-exposure, and the real mitigation in both cases is fixing the
underlying vulnerability class (sanitizing output for XSS, `SameSite`
cookies + anti-CSRF tokens for CSRF), not just picking a storage location
and calling it solved.

## Summary

- OAuth2 delegates authorization ("what can this client do"); OIDC adds
  identity/authentication on top of it; a JWT is just the token format
  commonly used to carry both.
- Default to authorization code (+ PKCE) for browser flows, client
  credentials for service-to-service; treat implicit and password grants as
  deprecated.
- A JWT's payload is encoded, not encrypted — never put secrets in it; the
  signature guarantees integrity, not confidentiality.
- RS256's asymmetric split (private key signs, public key verifies) is what
  lets many services safely verify tokens without any of them being able to
  mint one — HS256's shared secret can't offer that.
- Always validate `exp`, `iss`, and `aud`, not just the signature; rotate
  refresh tokens so reuse of a stolen one is detectable.
- Token storage is a trade-off, not a solved problem: `localStorage` risks
  XSS exposure, `httpOnly` cookies risk CSRF exposure — fix the underlying
  vulnerability class either way.

## Related Articles

- [AWS Cognito](../devops-tools/aws/cognito.md) — a concrete managed OAuth2/
  OIDC provider: User Pools issue exactly the ID/access/refresh token triple
  described here.
- [API Lifecycle & Design](api-lifecycle-design.md) — where auth fits into
  the broader contract a consumer relies on.
- [API Optimization & Resilience](api-optimization-resilience.md) — retry
  and idempotency patterns that apply to authenticated API calls too.
