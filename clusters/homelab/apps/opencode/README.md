# OpenCode

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
