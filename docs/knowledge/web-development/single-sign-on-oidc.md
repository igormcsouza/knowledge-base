---
tags:

- sso
- oidc
- oauth2
- authentication
- cognito
- web-development
- advanced

---

# Single Sign-On with OIDC: One Login, Many Apps

Single Sign-On (SSO) means a user signs in **once**, at one place, and every app that
trusts that place lets them in without asking again. The mechanism is not new
technology: it is [OIDC](oauth2-jwt-authentication.md#oidc-authentication-on-top-of-oauth2)
(authorization code + PKCE) where *several apps are clients of the same identity
provider (IdP)*. What makes it "single sign-on" is a **session at the IdP**, separate
from each app's own session.

## The Moving Parts

- **Identity provider (IdP)** — owns the users and the login page. Example: a
  [Cognito User Pool](../devops-tools/aws/cognito.md) with its hosted login domain.
- **App client** — one registration per app at the IdP, with its own `client_id`,
  allowed callback URLs, allowed logout URLs and scopes.
- **Two sessions** — the IdP's session (a cookie on the IdP's domain) and each app's
  session (tokens kept by the app). They are independent and expire separately.
- **Launcher / portal** — optional page that lists the apps and links to them. It is
  just another client; it does not "hold" the others' logins.

## What Happens on the Second App

```text
Open app A  -> no app session -> redirect to IdP /authorize -> no IdP session
            -> user types credentials once -> IdP sets its cookie
            -> redirect back to A with ?code=...  -> A exchanges it for tokens
Open app B  -> no app session -> redirect to IdP /authorize -> IdP cookie present
            -> IdP skips the login form -> redirect back to B with ?code=... instantly
```

The user sees B load already signed in, but under the hood B ran its own full code
exchange. Nothing is shared between A and B except the IdP cookie.

## Per-App Clients, Not One Shared Client

Give every app its **own** client at the IdP rather than sharing one:

- the **audience (`aud`) / `client_id`** in the tokens tells you which app a token
  was issued to, so one app cannot replay a token as another;
- callback and logout URLs are allow-listed per client, so a leaked client id from a
  low-stakes app cannot redirect codes to a different app;
- scopes, token lifetimes and secrets can differ (a public SPA client with PKCE and
  no secret, a server-side app as a confidential client with one);
- an app can be retired by deleting its client without touching the others.

## Identity Is the `sub`, Not the Username or Email

Each user has an immutable id (`sub`) per pool. Apps should key their data on it.
Two consequences:

- **Different pools mean different `sub`s for the same human.** Moving an app from its
  own user pool to a shared one changes every user's id, so stored data keyed by the
  old `sub` appears to vanish. Migrating means **copying/rewriting records from the old
  `sub` to the new one before switching** the app over, and verifying keys before
  deleting the old rows.
- Email and username are attributes, not identity; they can change.

## Where Claims Live

- The **ID token** carries profile claims (`name`, `picture`, `email`) for the app to
  display; the **access token** carries scopes and is meant for APIs. An app that
  wants a name or avatar reads it from the ID token.
- Profile attributes are only as good as what the IdP stores: a `picture` claim holds
  a URL the IdP never fetches, so the image has to be hosted somewhere public.
- Claims are a snapshot from the moment of login. Changing a user's name at the IdP
  shows up after the next sign-in or token refresh, not instantly.

## Sessions, Refresh and Sign-Out

- **App session** = the tokens the app holds. Access tokens are short; a refresh
  token (days to weeks) lets the app renew without involving the user, as long as the
  IdP session or refresh token has not been revoked.
- **Signing out of one app** clears only that app's tokens (and optionally calls the
  IdP's logout endpoint, which also ends the IdP session, so the *next* app visit
  asks for credentials again).
- **"Sign out everywhere"** means revoking the user's refresh tokens at the IdP
  (Cognito: global sign-out). Apps then fail their next refresh and fall back to
  login. Already-issued access tokens keep working until they expire, which is why
  they are short-lived.
- A portal's "active sessions" list reflects what the IdP knows (refresh tokens), not
  what a browser tab currently shows.

## Operational Lessons

- **Cache-bust the app shell.** A static frontend that hard-codes the IdP's client id
  and domain must be served with `Cache-Control: no-cache` for its HTML, otherwise
  users keep running the old build pointed at the old pool after a cutover.
- **Cut over one app at a time**, with data migrated first, and delete the old pool
  only after the user confirms sign-in works. Pools with a retain policy in IaC need
  manual deletion.
- **Keep deploy pipelines from re-bootstrapping** shared infrastructure; one app's CI
  run should not be able to alter the IdP or another app's resources.
- **Login domain is shared, the app is not.** The hosted login page lives on the IdP's
  domain for every app. The user tells apps apart by the client they are redirected
  back to, and an API (such as an MCP server) tells them apart by validating issuer,
  audience and client id of the token.
- **One-person identity hubs are fine.** For a personal set of apps, one pool with
  self-signup disabled and a single user is a complete SSO, with MFA optional.

## Summary

- SSO = one IdP session + each app doing its own OIDC code (+ PKCE) exchange; the IdP
  cookie is what skips the second login.
- One client per app: separate audiences, callback allow-lists and lifetimes.
- Key data on `sub`; moving between pools changes it, so migrate data before the
  cutover.
- Profile claims come in the ID token; sign-out semantics differ between "this app",
  "this app + IdP session" and "everywhere".
- Treat cutover as a deploy: migrate, cache-bust, verify, then delete the old pool.

## Related Articles

- [OAuth2 & JWT Authentication](oauth2-jwt-authentication.md) — grant types, PKCE,
  token validation and refresh rotation.
- [AWS Cognito](../devops-tools/aws/cognito.md) — User Pools as the IdP, app clients
  and hosted login.
- [MCP Authorization](mcp-authorization.md) — how an API with its own OAuth surface
  sits next to an SSO login and uses it for the consent step.
