# Authentication via forward-auth proxy (Authelia), not in-app

The app is internet-exposed for a non-technical Patient who logs from anywhere, so login must be low-friction yet strong. We put **Authelia in front as a Caddy `forward_auth` proxy with passkey (WebAuthn) login** and give the app **no login of its own** — it trusts `Remote-User`/`Remote-Groups` headers from the proxy and maps the `admins` group to the Admin role.

## Considered options
- **App-native auth (passkeys in-app):** full control of UX, but we'd build and maintain auth, sessions, and recovery ourselves.
- **Forward-auth proxy (chosen):** passkeys for free, only this app's subdomain is protected (rest of the homelab untouched), and a path to homelab-wide SSO later — at the cost of a generic login portal and the app trusting proxy headers.
- **Authentik:** a fuller IdP, but heavier (Postgres + Redis + worker) than needed to protect one app.

## Consequences
- The app must only ever be reachable *through* the proxy (never directly), or header trust is spoofable. Concretely: the app container publishes **no host port** (proxy-network only), Caddy sets `trusted_proxies`, the app **fails closed** when `Remote-User` is absent, and mutating/admin endpoints re-derive the role server-side. This is a fixed build acceptance criterion, not advisory.
- The SPA must handle session expiry gracefully — a background `fetch` can receive an auth redirect instead of a clean 401.
- Local dev fakes the identity headers via a shim (`DEV_USER`/`DEV_ROLE`) so Authelia isn't required for the daily loop.
