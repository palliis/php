# Palliis PHP FPM build and maintenance runbook

Keep this process with the Dockerfiles and publishing workflows in
[palliis/php](https://github.com/palliis/php). The filename is retained for existing
links; this guide covers both Trixie and Alpine. Last reviewed: 2026-10-21.

## Source and published images

The initial build used the unchanged Dockerfiles and helper scripts from
[docker-library/php PR #1694](https://github.com/docker-library/php/pull/1694),
at reviewed commit `f6048b5f6e3c2202ee082665a4ccc0d90de291a3`.
Forking upstream's default branch alone would not have included that unmerged PR.
PHP source checksums and GPG verification remain enabled. Application extensions
and configuration belong in downstream images.

The default and publishing branch is **`codex/fpm-trixie`**, including for Alpine.

| Full image tag | Docker build context | Platforms |
| --- | --- | --- |
| `ghcr.io/palliis/php:8.3.36-fpm-trixie` | `8.3/trixie/fpm` | linux/amd64, linux/arm64 |
| `ghcr.io/palliis/php:8.4.27-fpm-trixie` | `8.4/trixie/fpm` | linux/amd64, linux/arm64 |
| `ghcr.io/palliis/php:8.5.12-fpm-trixie` | `8.5/trixie/fpm` | linux/amd64, linux/arm64 |
| `ghcr.io/palliis/php:8.3.36-fpm-alpine3.23` | `8.3/alpine3.23/fpm` | linux/amd64, linux/arm64 |

These are Palliis builds, not Docker Official Images. Use the full PHP patch
version; no floating minor-version or `latest` aliases are published. A rebuild
can update Debian or Alpine packages under the same PHP tag. For an immutable
consumer, record and pin the multi-platform index digest as well as the tag.

## Rebuild the current versions in CI

Prerequisites: GitHub access to run Actions in `palliis/php`, and Docker Buildx
locally if inspecting the resulting manifests. No local Docker build is needed.

In the Actions tab, select **Publish PHP FPM Alpine** or **Publish PHP FPM Trixie**,
then **Run workflow** on `codex/fpm-trixie`. The equivalent commands are:

```sh
# Run the workflow for the images you need; these are independent builds.
gh workflow run publish-fpm-alpine.yml --repo palliis/php --ref codex/fpm-trixie
gh workflow run publish-fpm-trixie.yml --repo palliis/php --ref codex/fpm-trixie

gh run list --repo palliis/php --workflow publish-fpm-alpine.yml --limit 5
gh run list --repo palliis/php --workflow publish-fpm-trixie.yml --limit 5
```

Use the matching run ID with `gh run view RUN_ID --repo palliis/php` or
`gh run watch RUN_ID --repo palliis/php --exit-status`. Check its source commit and
the final publishing job, then inspect the tag:

```sh
docker buildx imagetools inspect ghcr.io/palliis/php:8.3.36-fpm-alpine3.23
docker buildx imagetools inspect ghcr.io/palliis/php:8.3.36-fpm-trixie
```

Confirm both `linux/amd64` and `linux/arm64`; provenance entries may also appear.
Save the source commit, workflow run URL, full tag, and index digest with the
downstream change. An architecture build alone does not prove the final tag was
published.

The workflows build on native `ubuntu-24.04` and `ubuntu-24.04-arm` runners. Each
architecture pushes by digest; the final job combines both into the full-version
tag. PR runs cannot publish. Source/workflow changes trigger the corresponding
push workflow; documentation-only pushes do not. Check for an existing push run
before dispatching manually. New workflows can briefly return 404 while GitHub
indexes them.

## Update to a new PHP release or reviewed upstream PR

Start from the maintained fork, preserving its publishing workflows:

```sh
git clone --config core.autocrlf=false --branch codex/fpm-trixie https://github.com/palliis/php.git php-palliis
cd php-palliis
git remote add upstream https://github.com/docker-library/php.git

# Historical example: substitute the PR number being reviewed for the next update.
git fetch upstream pull/1694/head
git rev-parse FETCH_HEAD
git show --stat FETCH_HEAD
```

1. Record and review the exact upstream commit. Work on a new `codex/` branch;
   merge or cherry-pick the selected upstream changes. Preserve generated
   Dockerfiles, helper scripts, source hashes, and signature verification. Keep
   the fork's publishing workflows and its upstream-only CI/templating guards.
   Preserve the README runbook link when updating upstream's generated README.
2. Match every published version to the target Dockerfile's `ENV PHP_VERSION`.
   The workflows do **not** derive tag versions automatically. Update:
   - [Trixie workflow](.github/workflows/publish-fpm-trixie.yml): both
     `jobs.build.strategy.matrix.php` and `jobs.publish.strategy.matrix.version`.
   - [Alpine workflow](.github/workflows/publish-fpm-alpine.yml): `env.PHP_VERSION`.
   - This image table and the reviewed-source record above.
3. If adding a PHP series or distro variant, also update build contexts, push/PR
   path filters, job names, cache scopes, artifact names/patterns, and tag suffixes.
   Changing the branch name requires updating the trigger filters and **every**
   publishing guard in both workflows. Publishing is restricted to `palliis/php`,
   `refs/heads/codex/fpm-trixie`, and non-PR events.
4. Review the diff and open a PR against `codex/fpm-trixie`. PR builds do not
   publish images. Merge the reviewed update, follow its publishing run, and
   inspect both platforms as described above.

This fork adds no smoke-test suite. Upstream source verification and compilation
checks remain intact; downstream applications retain their normal required CI.

## GHCR and Actions access

The `php` package is public as of the review date, so consumers can pull it
anonymously. Check visibility in the
[package settings](https://github.com/orgs/palliis/packages/container/php/settings).
New GHCR packages initially default to private; public repository visibility alone
does not make a package public. If keeping a package private, grant the consumer
repositories package access and configure their build authentication.

CI publishing uses the repository's `GITHUB_TOKEN` with `packages: write` and the
existing GHCR login step. No personal registry secret is needed for these jobs.
Local pushes require an account/token permitted to write this package.

The repository's Actions policy allows six actions at the exact commit SHAs used
in the workflows: `actions/checkout`, `docker/setup-buildx-action`,
`docker/login-action`, `docker/build-push-action`, `actions/upload-artifact`, and
`actions/download-artifact`. When updating a pin, update its matching repository
allowlist entry too; do not enable all actions to work around a rejected pin.
An inherited upstream workflow can fail its startup action-policy check even when
its job guard excludes this fork. Identify the workflow first; its result is
separate from the two Palliis publishing workflows.

```sh
gh api repos/palliis/php/actions/permissions/selected-actions
gh api orgs/palliis/packages/container/php --jq '{name,visibility}'
```

## Local emergency build on Windows

CI is the normal multi-platform publisher. The original Alpine image was built
locally for amd64 while CI was being prepared. To repeat that build, use the
reviewed Git tree as a binary tar context; this preserves executable permissions
and avoids Windows line-ending conversion.

Prerequisites: Docker Desktop in Linux-container mode, Buildx, Git, Python 3,
and an authenticated GitHub CLI token with permission to push packages. Run from
the fork checkout. Inspect builders and authenticate without printing the token:

```powershell
docker buildx ls
gh auth token | docker login ghcr.io --username (gh api user --jq .login) --password-stdin
```

The successful local builder was `desktop-linux`. Change it in this example if
your Linux builder has another name. The example repeats the initial PHP 8.3.36
Alpine build method; update the source, context, version, and tag variant together
for another release.

```powershell
@'
import subprocess

source_commit = "f6048b5f6e3c2202ee082665a4ccc0d90de291a3"
context_path = "8.3/alpine3.23/fpm"
php_version = "8.3.36"
image = f"ghcr.io/palliis/php:{php_version}-fpm-alpine3.23-local-amd64-{source_commit[:8]}"
context = subprocess.check_output([
    "git", "-c", "core.autocrlf=false", "archive",
    f"{source_commit}:{context_path}",
])
subprocess.run([
    "docker", "buildx", "build", "--builder", "desktop-linux",
    "--platform", "linux/amd64", "--pull", "--tag", image, "--push",
    "--label", "org.opencontainers.image.source=https://github.com/palliis/php",
    "--label", f"org.opencontainers.image.revision={source_commit}",
    "--label", f"org.opencontainers.image.version={php_version}",
    "-",
], input=context, check=True)
'@ | python -
```

Keep `-c core.autocrlf=false`: even `git archive` can apply CRLF conversion.
The initial attempt without it failed with `docker-php-source: not found` despite
the script being present. Python carries the tar as bytes; do not pipe that tar
through a PowerShell text pipeline.

Use a temporary local-build tag as above. Pushing a single-platform image to an
existing canonical tag replaces its multi-platform manifest and can break arm64
pulls. Let CI publish the canonical full-version tag after both native builds pass.

## Adopt the image and verify the deployed runtime

Update the application's Dockerfile, then rebuild and publish the application
image through its normal pipeline:

```dockerfile
FROM ghcr.io/palliis/php:8.3.36-fpm-alpine3.23
```

Publishing a PHP base image does not update running applications. Promote the
new application image through the intended environment's existing GitOps/release
flow. QA success does not establish that staging or production has been promoted.

Kubernetes pulls the finished application image, not its base image separately.
Existing GHCR image-pull secrets can continue to serve that application package;
changing the base alone does not require an Argo CD credential change. Private
application packages still need their own valid runtime pull credentials.

For verification, check the actual pod image digest, readiness, and migration/init
result. Where pod exec is authorized, run `php -v` inside each PHP container and
`php-fpm -v` inside FPM containers. To identify the binary of a running process,
inspect `readlink /proc/1/exe`; if it resolves to PHP/FPM, `/proc/1/exe -v` checks
that executable directly. If PID 1 is a supervisor, identify the PHP process first.
Use explicitly selected cluster credentials; read-only observer access may deny
pod exec and must not be silently replaced with administrator credentials.

Shared-hosting PHP is managed separately by the hosting provider. Container-image
updates do not change that runtime. To roll back a deployment, promote the previous
known application image tag/digest through the same release flow; do not republish
an old PHP binary under a newer version tag.

## Initial build record

- Source: [upstream PR #1694](https://github.com/docker-library/php/pull/1694),
  commit `f6048b5f6e3c2202ee082665a4ccc0d90de291a3`.
- [Successful Trixie publishing run](https://github.com/palliis/php/actions/runs/39616381883).
- [Successful Alpine publishing run](https://github.com/palliis/php/actions/runs/39618668084).
- Alpine multi-platform index from that run:
  `sha256:d36cd8c0a27ae7fe0ad31e46993a570b8b8aedbff9137923c49d88711586ba3d`.

This is historical evidence, not an assertion that mutable tags still point at
those builds. Inspect the registry again before each rollout.
