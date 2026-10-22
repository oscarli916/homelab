# OpenCode

## Tailscale authentication

The original Tailscale hostname now uses the tested Authentik proxy route:

| URL | Route | Authentication |
| --- | --- | --- |
| `https://opencode.tail5e11fb.ts.net` | Authentik embedded proxy outpost | `authentik Admins` only |
| `https://opencode.home.arpa` | Existing direct LAN route | None, unchanged |
| `https://opencode-auth.tail5e11fb.ts.net` | Retired trial route | Removed |

The authenticated route uses the same OpenCode instance and data. No OpenCode
deployment, Service, or volume changes are needed. The Tailscale ingress is
managed under `infrastructure/authentik`. If authentication is misconfigured or
Authentik is unavailable, use the LAN URL from your home network. LAN access
still works without Tailscale connected.

OpenCode is a shared workspace with shell access and stored credentials, not an
isolated workspace per user. The LAN route remains an intentional authentication
bypass, including for remote users who can reach it through subnet routing.

### Authentik provider settings

These objects are configured in Authentik's admin UI, not provisioned by these
manifests. Sign in at `https://authentik.tail5e11fb.ts.net` and open the admin UI.

1. Under **Applications > Providers**, reuse the tested **Proxy Provider**
   `OpenCode Tailscale Trial`. Keep **Proxy** mode, not forward auth. Its name
   can optionally be changed to `OpenCode Tailscale`.
2. Set **External host** to `https://opencode.tail5e11fb.ts.net` and
   **Internal host** to `http://opencode-web.opencode.svc.cluster.local:3000`.
   Use the default implicit-consent authorization flow and default provider
   invalidation flow. Leave unauthenticated paths empty and HTTP Basic
   authentication disabled.
3. Keep the tested application and its provider association. Its display name
   can optionally be changed to `OpenCode Tailscale`; the existing slug can stay.
   Update its launch URL if it explicitly references the trial hostname.
4. On the application's **Policy / Group / User Bindings** tab, bind the existing
   `authentik Admins` group. Do not leave the application without bindings, which
   would allow other authenticated users. Remove any broader access bindings.
5. Under **Applications > Outposts**, keep the OpenCode application assigned to
   **authentik Embedded Outpost**, preserving other assigned applications.
   Check `authentik_host` is `https://authentik.tail5e11fb.ts.net/`. If
   `authentik_host_browser` is set, use the same Tailscale URL there. This changes
   the login URL for all applications on this outpost; review existing users of
   the outpost before changing it. Future LAN authentication without Tailscale
   will need a LAN-reachable login URL and a separate routing configuration.
   Preserve the working login settings from the trial. Do not change unrelated
   providers or shared outpost settings as part of this cutover.
6. Confirm that the outpost is healthy and has loaded the provider before
   deploying the ingress change.

No additional outpost deployment or outpost API token is needed. The embedded
outpost runs in the existing Authentik server. The Tailscale ingress lives in the
`authentik` namespace because an Ingress backend Service must be in the same
namespace. The LAN ingress remains in `opencode`.

### Cut over from the trial

Moving the original hostname between ingress namespaces creates a new Tailscale
proxy device. Plan a short interruption to Tailscale access. Perform this from
the LAN using `https://opencode.home.arpa` and a terminal with cluster access,
not solely through the Tailscale OpenCode session being replaced.

1. Before merging the cutover, suspend both affected Flux Kustomizations. Ensure
   they remain suspended until the old hostname is released:

```sh
flux suspend kustomization apps
flux suspend kustomization infrastructure-authentik
```

2. Change the tested provider's external host to
   `https://opencode.tail5e11fb.ts.net` as described above. Preserve the existing
   application, admin binding, and outpost assignment. Wait for the outpost to
   load the update. The trial URL stops working during this step.
3. Delete only the old direct Tailscale ingress. Do not delete the OpenCode
   Service, deployment, LAN ingress, or volumes:

```sh
kubectl -n opencode delete ingress opencode-tailscale
```

4. Wait for the old proxy resources to be cleaned up and the old `opencode`
   device/name to be released in Tailscale. If an orphaned device remains, verify
   it belongs to the deleted ingress before removing it in the Tailscale admin
   console. Do not remove the Authentik or trial devices to release this name.
5. Merge the cutover and fetch it, then reconcile both Kustomizations:

```sh
flux reconcile source git flux-system
flux resume kustomization apps
flux reconcile kustomization apps --with-source
flux resume kustomization infrastructure-authentik
flux reconcile kustomization infrastructure-authentik --with-source
kubectl -n authentik get ingress opencode-tailscale
```

6. Verify the assigned hostname is exactly `opencode.tail5e11fb.ts.net`. If a
   suffixed name appears, stop and resolve the old device/name conflict before
   testing. Flux prunes the old `opencode-auth-tailscale` ingress and its trial
   device. Do not delete the reused Authentik application/provider.

### Verify authentication

1. In a private browser window connected to Tailscale, open
   `https://opencode.tail5e11fb.ts.net`. It must redirect to Authentik at
   `authentik.tail5e11fb.ts.net`, not `authentik.home.arpa`.
2. Sign in as an `authentik Admins` member. Confirm OpenCode loads, messages
   stream, and the terminal connects. These exercise streaming and WebSockets
   through the proxy.
3. In a separate private session, sign in as a non-admin user. Access must be
   denied. Do not rely only on a successful admin login to validate the binding.
4. Open `https://opencode.tail5e11fb.ts.net/outpost.goauthentik.io/sign_out`,
   then confirm the app requires authentication again. An existing
   Authentik SSO session may log you back in without another password prompt;
   use a fresh private session to verify the unauthenticated redirect.
5. On the LAN with Tailscale disconnected, open `https://opencode.home.arpa`.
   It must still load directly without an Authentik login.
6. Confirm the old trial ingress is gone and the trial URL no longer serves
   OpenCode. Confirm `kubectl -n opencode get ingress` lists no direct Tailscale
   ingress.

### Roll back

Use LAN access while fixing the provider or routing. To restore the direct
Tailscale route, revert the cutover in Git and change the provider external host
back to `https://opencode-auth.tail5e11fb.ts.net`. Suspend both Kustomizations,
delete the new `authentik/opencode-tailscale` ingress, and wait for its `opencode`
device/name to be released before reconciling the reverted manifests. Restoring
the original manifests without releasing that name can cause the same hostname
collision. No OpenCode data restoration is needed. Leave the LAN ingress intact.

## Build and deploy

The custom image includes GitHub CLI, Git, SSH, and the PTY library used by the
OpenCode terminal. GitHub Actions builds it using
`.github/workflows/build-images.yaml`. See `.github/README.md` for registry setup
and the image allowlist.

When changing the Dockerfile, choose a new tag and update both
`.github/images.json` and `deployment.yaml` in the same PR. The workflow tests the
build on the PR and publishes on merge to `main`. Flux may temporarily report an
image pull error while the build runs, then recover after the image is published.
The workflow does not commit deployment changes or restart pods itself.

The image currently builds for `linux/amd64`. The downloaded PTY library is
architecture-specific, so supporting ARM also requires changing that download.

## Skills and commands

Each new pod fetches the latest commit on the dotfiles repository's default branch
and copies its skills and commands. The init container logs the loaded commit SHA.
Push changes to `oscarli916/dotfiles`, then replace the pod to refresh them:

```sh
kubectl -n opencode rollout restart deployment/opencode
```

Running pods do not refresh automatically. No image rebuild is needed. If the
sync fails or exceeds 90 seconds, the pod starts without synced skills or commands.

## One-time GitHub login

After the new image is deployed, run this from a machine with cluster access:

```sh
kubectl -n opencode exec -it deployment/opencode -c opencode -- \
  gh auth login --hostname github.com --git-protocol ssh --skip-ssh-key --web --insecure-storage
```

Alternatively, run the `gh auth login` command directly in OpenCode's terminal.
Open the URL shown by the CLI in your own browser and enter its one-time code.
`--skip-ssh-key` keeps the existing SSH key setup unchanged.

GitHub CLI stores its config and token in `/root/.config/gh`, mounted from the
`gh` subdirectory of the existing `opencode-data` PVC. The login survives container
restarts, pod replacements, and image updates as long as that PVC and its data
are retained. Reauthenticate if the token is revoked or becomes invalid.

`--insecure-storage` deliberately stores the token as plaintext on the volume
rather than in an ephemeral container keyring. Treat the volume and its backups
as credentials. Anyone able to run commands in OpenCode can use this GitHub
account's granted permissions. Never copy the token or `hosts.yml` into Git.

## Verify

Check authentication and PR access after login:

```sh
kubectl -n opencode exec deployment/opencode -c opencode -- gh auth status
kubectl -n opencode exec deployment/opencode -c opencode -- \
  gh pr list --repo oscarli916/homelab
```

To verify persistence, replace the pod and check authentication again:

```sh
kubectl -n opencode rollout restart deployment/opencode
kubectl -n opencode rollout status deployment/opencode
kubectl -n opencode exec deployment/opencode -c opencode -- gh auth status
```
