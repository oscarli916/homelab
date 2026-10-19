# OpenCode v2

Runs OpenCode 2.0.32 alongside v1. The existing `apps` Flux Kustomization
continues to manage v1; `apps-opencode-v2` manages this directory separately.
Do not add this directory to `apps/kustomization.yaml` as well.

## Isolation

- Uses the existing `opencode` namespace and `opencode-secrets` Secret.
- Uses new `opencode-v2-workspace` and `opencode-v2-data` Longhorn PVCs.
- Clones the same four repositories into its own workspace.
- Loads skills and commands from the same pinned dotfiles revision as v1.
- Does not copy or mount v1 configuration, provider credentials, sessions, or workspace data.
- Depends on the existing `apps` Flux Kustomization for the namespace and SealedSecret.

V1 resources and storage are unchanged. V2 starts with fresh provider connections;
connect your providers again in the v2 interface. Git SSH credentials and the
server password are shared, so changing those shared credentials affects both versions.
The clone container prepares a pod-local SSH directory with the key and GitHub
host keys; the server mounts that directory read-only for later Git operations.

## Image

Build and push the custom image before merging the manifests to `main`:

```sh
docker build -t oscarli916/opencode-custom:2.0.32 clusters/homelab/apps/opencode-v2
docker push oscarli916/opencode-custom:2.0.32
```

Build for your cluster nodes' architecture. The Dockerfile retains Git and SSH
from the v1 customization. It omits the v1-specific `bun-pty` shared library;
verify the integrated terminal when testing v2.

V2 uses `serve` instead of `web`, still on port 3000. The server runs in the
foreground, without a background service or public OpenTunnel access.

## Access

- LAN: `https://opencode-v2.home.arpa`
- Tailscale: the HTTPS URL provisioned for the `opencode-v2` hostname
- Username: `opencode`, which v2 fixes independently of the v1 username
- Password: the existing `opencode-secrets` `server-password` value

Ensure LAN DNS resolves `opencode-v2.home.arpa` to Traefik, using the same
mechanism as `opencode.home.arpa`. The v2 Ingress requests its own TLS certificate.

## Validation

```sh
kubectl kustomize clusters/homelab/apps/opencode-v2
```

After the image is published and Flux deploys the manifests, check the pod,
both ingress routes, provider login, skills, commands, Git SSH access, and the
integrated terminal. Confirm v1 remains accessible at its original URLs.
