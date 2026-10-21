# Grafana access

Grafana has two ingresses pointing to the same instance:

| URL | Login method |
| --- | --- |
| `https://grafana.home.arpa` | Grafana username and password, without requiring Tailscale |
| `https://grafana.tail5e11fb.ts.net` | Grafana username and password or Authentik SSO, with Tailscale connected |

Password login requires a Grafana local account. An Authentik account and its
password do not automatically provide Grafana password login.

The canonical `root_url` is the Tailscale URL, so OAuth callbacks and generated
absolute links use that hostname. Domain enforcement and automatic SSO are
disabled to preserve password login through the LAN hostname. Start SSO from
the Tailscale URL, not the LAN URL, so the OAuth state cookie and callback use
the same hostname.

Only the browser-facing authorization URL uses Authentik's Tailscale hostname.
Grafana's token and user-info requests continue through `authentik.home.arpa`,
using cluster DNS and the mounted homelab CA certificate. Pods do not need
Tailscale connectivity for those requests.

Signing out ends the Grafana session without redirecting to Authentik. The
Authentik session remains active, so a subsequent SSO login may not ask for
credentials again. This avoids sending LAN password users to a Tailscale-only
logout URL.

## Before rollout

Authentik's provider settings are stored in its database, not managed by these
manifests. Before merging the Grafana configuration change:

1. Verify a Grafana local account can sign in at `https://grafana.home.arpa`.
2. Open Authentik's admin interface and edit the Grafana OAuth2/OIDC provider.
3. Add a **Strict** redirect URI of `https://grafana.tail5e11fb.ts.net/login/generic_oauth`.
4. Keep the existing LAN redirect URI until rollout and verification finish, so
   the previous configuration remains usable if rolled back.
5. If the Grafana application has a launch URL, set it to `https://grafana.tail5e11fb.ts.net`.

Do not change the provider's client ID or secret. Both existing ingresses and
their TLS configurations remain unchanged. Flux deploys the manifest changes
after they are merged to `main`.

## Verification

1. On the home network with Tailscale disconnected, open `https://grafana.home.arpa`,
   sign in with a Grafana password, load a dashboard, and sign out. Trust the
   homelab root CA on the client as before.
2. Outside the home network with Tailscale connected, open
   `https://grafana.tail5e11fb.ts.net` and test Grafana password login and logout.
3. In a fresh browser session on the Tailscale URL, select Authentik and verify
   authorization uses `authentik.tail5e11fb.ts.net` and returns to
   `grafana.tail5e11fb.ts.net/login/generic_oauth`. Verify a dashboard loads.
4. Verify the expected Grafana role is assigned and sign out. An Authentik
   session remaining active is expected.

If SSO reports an invalid redirect URI, check the provider allowlist. If the
callback reports a missing state cookie, start again from the Tailscale Grafana
URL. Token or user-info failures require checking Grafana's in-cluster DNS,
connectivity, and CA trust for `authentik.home.arpa`.
