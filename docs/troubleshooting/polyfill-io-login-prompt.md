---
tags:

- troubleshooting
- security
- mkdocs
- web-development

---

# Browser Keeps Prompting to Log In on Every Article (polyfill.io)

Every page load was triggering a native browser login (Basic/NTLM auth) dialog, traced
back to a third-party script loaded site-wide from the compromised `polyfill.io` domain.

## Environment

- Tool/library + version: MkDocs Material, site-wide `extra_javascript`
- OS / runtime: Any browser, any OS — reproducible on every page of the deployed site
- Any other relevant context: `mkdocs.yml` loaded
  `https://polyfill.io/v3/polyfill.min.js?features=es6` as a global script

## Problem

Opening any article on the site (not just one specific page) popped a native browser
"Sign in" / HTTP authentication dialog, unrelated to anything in the site's own content
or backend. It happened consistently, on every navigation, because the offending script
was wired into the site config globally rather than on a single page.

## Root Cause

In February 2024, the `polyfill.io` domain and its GitHub org were sold to Funnull, a
company later linked (via a North Korea-connected infostealer investigation) to a
supply-chain attack. Sites that loaded `cdn.polyfill.io`'s script were served malicious,
dynamically-generated payloads based on request headers — redirecting mobile users to
gambling sites used for crypto laundering. Namecheap suspended the original domain on
June 27, 2024, but it has since been resold/parked. The odd behavior now seen (a native
browser auth dialog) is consistent with the parked domain returning `401 Unauthorized`
with a `WWW-Authenticate` header instead of the JS payload it used to serve — browsers
show a login prompt automatically for that response, with no way for site code to
suppress it.

This repo's `mkdocs.yml` had:

```yaml
extra_javascript:
  - javascripts/mathjax.js
  - https://polyfill.io/v3/polyfill.min.js?features=es6
  - https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js
```

`extra_javascript` entries are injected into every rendered page, so the compromised
script ran (and prompted) on every article, not just one.

## Fix

Remove the `polyfill.io` line entirely. All browsers MkDocs Material supports have
shipped native ES6 support since ~2017, so the polyfill wasn't doing anything useful
even before the domain turned malicious — this isn't a "find a safer CDN" situation, it's
dead code that happened to become a security liability.

```diff
 extra_javascript:
   - javascripts/mathjax.js
-  - https://polyfill.io/v3/polyfill.min.js?features=es6
   - https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js
```

If a project genuinely still needs ES6 polyfills for legacy browser support, self-host
the specific polyfills needed (e.g. via `core-js` bundled at build time) instead of
pointing at any third-party CDN for this purpose.

## Prevention

- Never point `extra_javascript` (or any script tag) at `polyfill.io` /
  `cdn.polyfill.io` — the domain has been flagged as malicious by Namecheap, Google,
  Cloudflare, and Fastly since mid-2024, and remains dangerous regardless of who
  currently owns it.
- Prefer self-hosting third-party scripts (or vendoring them at build time) over
  pointing at external CDNs for anything injected site-wide — a compromised CDN entry
  in a global config affects every page at once, which is exactly what happened here.
- When a browser shows an unexpected login/auth dialog on a site with no login feature,
  check the network tab for third-party script/asset requests before assuming it's a
  backend issue — it's often an external resource, not the app itself.
