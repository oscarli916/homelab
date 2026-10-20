# Container image builds

## Registry setup

Add these GitHub Actions repository secrets under Settings > Secrets and
variables > Actions:

- `DOCKERHUB_USERNAME`: the Docker Hub account that can push the listed images.
- `DOCKERHUB_TOKEN`: a Docker Hub access token with write permission.

The workflow publishes to Docker Hub, matching the existing Kubernetes image
references. It does not need cluster credentials. PR builds do not log in or push,
so forks do not receive registry credentials.

## Triggers and selection

The workflow starts only when a file named `Dockerfile` or `Dockerfile.*` changes
in a PR targeting `main` or a push to `main`. Docs, deployment manifests, and
allowlist-only changes do not trigger it.

`.github/images.json` is the explicit allowlist. Each entry sets its build
context, Dockerfile, Docker Hub image, tag, and platforms. The selection job
compares the push's before/after commits, or the PR's merge-base/head commits, and builds
only entries whose Dockerfile changed. Unlisted Dockerfiles publish nothing.
An unlisted Dockerfile change can still start the small selection job, but the
build job is skipped if nothing matches.

PRs test-build the selected images without pushing. A merge to `main` builds and
publishes them. There is no scheduled or manual all-images build. To retry a
failed build, rerun the original workflow from the Actions tab.

## Changing an image

1. Edit its Dockerfile.
2. Choose a new tag in `.github/images.json` and update the workload's image
   reference to the same tag in the same PR. Do not reuse a deployed tag.
3. Open the PR and check the build result.
4. Merge to `main`; Actions publishes the image and Flux deploys the manifest.

Flux and the image build run independently after merge. Flux may try pulling
before publication finishes and retry until the image exists. A failed publish
leaves the new workload unable to pull its image until the build is fixed and
rerun. This workflow automates publishing, not release ordering.

To add an image, add its entry to `.github/images.json` in the same PR that adds
its Dockerfile. New entries must use Dockerfile names matching the trigger above.
Choose platforms supported by the base image and any downloaded binaries.

Only Dockerfile changes trigger builds. If an image later uses `COPY` or `ADD`
with local files, changes to those files alone will not rebuild it. Extend the
trigger and selection logic before relying on such build inputs. Base-image or
package updates also do not trigger builds unless the Dockerfile changes.
