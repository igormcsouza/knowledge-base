---
tags:

- aws
- cognito
- authentication
- identity

---

# AWS Cognito

Cognito is AWS's managed identity service. It covers two distinct jobs that
are easy to conflate because they share a product name: **User Pools**
(who is this user, and are they who they say they are) and **Identity
Pools** (given a verified identity, what temporary AWS permissions should
they get).

## User Pools: Authentication

A User Pool is a managed user directory — it stores accounts, handles
sign-up/sign-in, password policies, MFA, and email/phone verification, and
speaks standard OAuth2/OIDC flows (including a hosted, customizable login
UI so an app doesn't need to build its own auth screens).

A successful sign-in returns three JWTs:

- **ID token** — claims about who the user is (email, sub, custom
  attributes) — what an application typically reads to know who's logged in.
- **Access token** — used to authorize calls to an API on the user's behalf,
  scoped to what the user pool grants.
- **Refresh token** — used to obtain new ID/access tokens without the user
  logging in again, until it expires.

## Verifying Tokens

A backend receiving one of these tokens (e.g. as a `Bearer` header) must
verify its signature against Cognito's public JWKS endpoint, and check the
issuer, audience, and expiry before trusting any claim inside it — a token
that merely decodes without error is not the same as a token that's valid.
In practice this verification step is usually handled for you: an API
Gateway **Cognito authorizer** validates the token before a request ever
reaches the backend/Lambda, so protected endpoints don't need custom
auth-checking code at all.

## Identity Pools: Temporary AWS Credentials

An Identity Pool is a separate concept: given a verified identity (a Cognito
User Pool token, or an external identity provider like Google/Facebook, or
even an unauthenticated "guest" identity), it exchanges that identity — via
AWS STS — for **temporary AWS credentials**. Those credentials let a client
call AWS services **directly**, scoped by an IAM role tied to the identity.

The canonical use case: a mobile or web client uploading a file straight to
an [S3](s3.md) bucket without routing the upload through a backend server at
all — the client gets temporary, scoped-down S3 credentials from an Identity
Pool and uploads directly.

## One Pool, Many Apps (SSO)

A single User Pool can serve any number of apps: each gets its own **app client**
(own `client_id`, callback/logout URLs and scopes) and all of them share the pool's
**hosted login domain**. Signing in to one app sets a session cookie on that domain,
so the next app's redirect skips the login form — that is
[SSO](../../web-development/single-sign-on-oidc.md). Things worth remembering:

- **`sub` is per pool.** The same person has a different `sub` in a different pool,
  so moving an app between pools means migrating the data keyed by it.
- **`name` and `picture` come in the ID token**, from the user's attributes; `picture`
  is only a URL, so the image must be hosted elsewhere.
- **Self-signup can be off** for a private pool; users are created by an admin.
- **No Dynamic Client Registration.** Cognito cannot register clients on the fly, so it
  cannot be the authorization server for a "paste a link" remote
  [MCP connector](../../web-development/mcp-authorization.md) without a shim or
  pre-registered client.
- **Pools created with a retain removal policy** are not deleted when the stack is;
  delete them by hand once nothing uses them.
- **Global sign-out** revokes the user's refresh tokens; access tokens already issued
  stay valid until they expire.

## When to Use Which

- Need "is this user logged in, and who are they" for your own API? →
  **User Pool**, verified via an API Gateway authorizer or manual JWT
  verification.
- Need a client to call AWS services (S3, DynamoDB, etc.) directly, without
  round-tripping through your backend? → **Identity Pool**, typically
  federated from a User Pool identity.

Many applications use both together: a User Pool authenticates the user, and
an Identity Pool converts that authentication into scoped AWS credentials
for the specific AWS resources the client needs to touch directly.

## Summary

- User Pools handle authentication: accounts, login flows, and issuing
  JWTs.
- Identity Pools handle authorization to AWS itself: exchanging a verified
  identity for temporary, IAM-role-scoped AWS credentials.
- Token verification (signature, issuer, audience, expiry) is what actually
  makes a JWT trustworthy — an API Gateway Cognito authorizer usually
  handles this automatically.
- The two are commonly chained: authenticate via User Pool, then federate
  into an Identity Pool for direct AWS access.

## Related Articles

- [S3](s3.md) — the most common target for Identity Pool credentials, e.g.
  direct client-side uploads.
- [Single Sign-On with OIDC](../../web-development/single-sign-on-oidc.md) — several
  apps as clients of one pool.
- [MCP Authorization](../../web-development/mcp-authorization.md) — why a provider
  without Dynamic Client Registration complicates remote MCP connectors.
