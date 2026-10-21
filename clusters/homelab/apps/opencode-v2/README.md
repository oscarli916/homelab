# OpenCode v2

Runs OpenCode 2.0.32 alongside v1. The existing `apps` Flux Kustomization
continues to manage v1; `apps-opencode-v2` manages this directory separately.
Do not add this directory to `apps/kustomization.yaml` as well.

## Isolation

- Uses the existing `opencode` namespace and `opencode-secrets` Secret.
- Uses new `opencode-v2-workspace` and `opencode-v2-data` Longhorn PVCs.
- Clones the same four repositories into its own workspace.
- Loads skills and commands from the latest dotfiles default-branch commit at pod startup, like v1.
- Does not copy or mount v1 configuration, provider credentials, sessions, or workspace data.
- Depends on the existing `apps` Flux Kustomization for the namespace and SealedSecret.

V1 storage remains separate. V2 starts with fresh provider connections;
connect your providers again in the v2 interface. Git SSH credentials come from
the existing Secret. V2 uses its `server-password` value; v1 no longer requires
server authentication. Changing the shared SSH key affects both versions.
The clone container prepares a pod-local SSH directory with the key and GitHub
host keys; the server mounts that directory read-only for later Git operations.

## Image

GitHub Actions builds `oscarli916/opencode-custom:2.0.32-gh` using the v2 entry
in `.github/images.json`. PRs test-build without pushing; merging to `main`
publishes the image for `linux/amd64`. See `.github/README.md` for registry setup.

Flux and Actions run independently after merge. Flux may report image pull errors
until publication finishes, then recover. If publication fails, fix the build and
rerun the workflow. No manual Docker build is required.

When changing the Dockerfile, choose a new tag and update both
`.github/images.json` and `deployment.yaml` in the same PR.

The Dockerfile includes GitHub CLI, Git, and SSH. It omits the v1-specific
`bun-pty` shared library; verify the integrated terminal when testing v2.

V2 uses `serve` instead of `web`, still on port 3000. The server runs in the
foreground, without a background service or public OpenTunnel access.

## Skills and commands

Each new pod fetches the latest commit on the dotfiles repository's default branch.
Push skill or command changes to `oscarli916/dotfiles`, then refresh v2 with:

```sh
kubectl -n opencode rollout restart deployment/opencode-v2
```

The init container logs the loaded commit SHA. Running pods do not refresh
automatically, and no image rebuild is needed. If sync fails or exceeds 90 seconds,
the pod starts without synced skills or commands.

## One-time GitHub login

V2 uses its own login, stored on `opencode-v2-data`, not v1's credentials:

```sh
kubectl -n opencode exec -it deployment/opencode-v2 -c opencode -- \
  gh auth login --hostname github.com --git-protocol ssh --skip-ssh-key --web --insecure-storage
kubectl -n opencode exec deployment/opencode-v2 -c opencode -- gh auth status
```

Open the URL printed by the CLI in your browser and enter its one-time code.
The login persists in `/root/.config/gh` across pod replacements and image updates.
`--insecure-storage` stores the token as plaintext on the PVC. Treat the volume
and backups as credentials, and never commit the token or `hosts.yml`.

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
