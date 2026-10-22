# OpenCode

## Tailscale authentication

Test Authentik through a separate hostname before removing any existing access:

| URL | Route | Authentication |
| --- | --- | --- |
| `https://opencode.tail5e11fb.ts.net` | Existing direct Tailscale route | None, unchanged |
| `https://opencode.home.arpa` | Existing direct LAN route | None, unchanged |
| `https://opencode-auth.tail5e11fb.ts.net` | New route through Authentik | `authentik Admins` only |

All three routes use the same OpenCode instance and data. No OpenCode deployment,
Service, or volume changes are needed. The new ingress is managed under
`infrastructure/authentik`, independently of the existing apps Kustomization.
If authentication is misconfigured or Authentik is unavailable, use either
original URL. LAN access still works without Tailscale connected.

OpenCode is a shared workspace with shell access and stored credentials, not an
isolated workspace per user. During this trial, the original unauthenticated
routes remain intentional bypasses. This does not secure all access to OpenCode.

### Configure Authentik before deploying

These objects are configured in Authentik's admin UI, not provisioned by these
manifests. Sign in at `https://authentik.tail5e11fb.ts.net` and open the admin UI.

1. Under **Applications > Providers**, create a **Proxy Provider** named
   `OpenCode Tailscale Trial`. Select **Proxy** mode, not forward auth.
2. Set **External host** to `https://opencode-auth.tail5e11fb.ts.net` and
   **Internal host** to `http://opencode-web.opencode.svc.cluster.local:3000`.
   Use the default implicit-consent authorization flow and default provider
   invalidation flow. Leave unauthenticated paths empty and HTTP Basic
   authentication disabled.
3. Under **Applications > Applications**, create an application named
   `OpenCode Tailscale Trial`, with slug `opencode-tailscale-trial`, and select
   that provider.
4. On the application's **Policy / Group / User Bindings** tab, bind the existing
   `authentik Admins` group. Do not leave the application without bindings, which
   would allow other authenticated users. Remove any broader access bindings.
5. Under **Applications > Outposts**, edit **authentik Embedded Outpost** and add
   the OpenCode application, preserving any applications already selected.
   Check `authentik_host` is `https://authentik.tail5e11fb.ts.net/`. If
   `authentik_host_browser` is set, use the same Tailscale URL there. This changes
   the login URL for all applications on this outpost; review existing users of
   the outpost before changing it. Future LAN authentication without Tailscale
   will need a LAN-reachable login URL and a separate routing configuration.
   If this shared setting would disrupt existing applications, stop and use a
   dedicated outpost for the trial instead. Do not change existing providers or
   point any provider at the original `opencode.tail5e11fb.ts.net` hostname.
6. Confirm that the outpost is healthy and has loaded the provider before
   deploying the ingress change.

No additional outpost deployment or outpost API token is needed. The embedded
outpost runs in the existing Authentik server. The trial ingress lives in the
`authentik` namespace because an Ingress backend Service must be in the same
namespace. The LAN ingress remains in `opencode`.

### Deploy the parallel route

Do not delete or move the original ingress. After this change is merged, let Flux
reconcile normally, or request reconciliation from a machine with cluster access:

```sh
flux reconcile source git flux-system
flux reconcile kustomization infrastructure-authentik --with-source
kubectl -n authentik get ingress opencode-auth-tailscale
kubectl -n opencode get ingress opencode-tailscale
```

Verify the new assigned hostname is `opencode-auth.tail5e11fb.ts.net` and the
original remains `opencode.tail5e11fb.ts.net`. The new name must be unused in
your tailnet; if Tailscale assigns a suffixed name, resolve the name conflict
before testing. No suspension of apps reconciliation or device migration is
needed. Until the provider is configured, the trial route will not proxy OpenCode;
the original routes are unaffected.

### Verify authentication

1. In a private browser window connected to Tailscale, open
   `https://opencode-auth.tail5e11fb.ts.net`. It must redirect to Authentik at
   `authentik.tail5e11fb.ts.net`, not `authentik.home.arpa`.
2. Sign in as an `authentik Admins` member. Confirm OpenCode loads, messages
   stream, and the terminal connects. These exercise streaming and WebSockets
   through the proxy.
3. In a separate private session, sign in as a non-admin user. Access must be
   denied. Do not rely only on a successful admin login to validate the binding.
4. Open `https://opencode-auth.tail5e11fb.ts.net/outpost.goauthentik.io/sign_out`,
   then confirm the trial app requires authentication again. An existing
   Authentik SSO session may log you back in without another password prompt;
   use a fresh private session to verify the unauthenticated redirect.
5. On the LAN with Tailscale disconnected, open `https://opencode.home.arpa`.
   It must still load directly without an Authentik login.
6. Reconnect Tailscale and confirm `https://opencode.tail5e11fb.ts.net` still
   works directly without an Authentik login.

### Remove the trial or complete the cutover

To remove the trial, remove `ingress-opencode-tailscale.yaml` from Authentik's
`kustomization.yaml` and delete that manifest in Git. Flux will prune only the
trial ingress. Then unassign the trial application from the embedded outpost and
delete its application/provider in Authentik. Do not delete the existing ingresses.

Only after the checks above pass and you approve the cutover, remove the original
Tailscale ingress and its resource entry in a separate change. You can keep
`opencode-auth.tail5e11fb.ts.net` as the authenticated address. Reusing the original
`opencode` hostname instead requires a planned device/hostname migration and an
updated provider external host; it is not part of this trial. Leave the LAN route
in place until LAN authentication without Tailscale has been tested separately.

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
