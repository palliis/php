# Palliis PHP FPM images

This fork builds the unchanged Dockerfiles from
[docker-library/php PR #1694](https://github.com/docker-library/php/pull/1694),
commit `f6048b5f6e3c2202ee082665a4ccc0d90de291a3`.
PHP source checksums and GPG verification remain enabled.

The `codex/fpm-trixie` branch publishes these tags to GHCR:

| Image | Platforms |
| --- | --- |
| `ghcr.io/palliis/php:8.3.36-fpm-trixie` | linux/amd64, linux/arm64 |
| `ghcr.io/palliis/php:8.4.27-fpm-trixie` | linux/amd64, linux/arm64 |
| `ghcr.io/palliis/php:8.5.12-fpm-trixie` | linux/amd64, linux/arm64 |
| `ghcr.io/palliis/php:8.3.36-fpm-alpine3.23` | linux/amd64, linux/arm64 |

There are no floating minor-version or `latest` tags. These are Palliis builds,
not Docker Official Images. Patch tags identify the PHP version; use an image
digest when an immutable base is required, since a rebuild can update Debian or Alpine
packages underneath the same PHP version.

Each architecture builds on a native GitHub runner. Successful builds are pushed
by digest; each workflow's final tags combine both architectures after its builds pass.
Pull requests only build and cannot publish images. This fork adds no smoke-test
suite; the original Dockerfiles retain their source verification and build checks.

To rebuild, run **Publish PHP FPM Trixie** or **Publish PHP FPM Alpine** from the
Actions tab on the `codex/fpm-trixie` branch. Alpine has its own workflow, so an
Alpine rebuild does not rebuild the Trixie images. The Alpine tag was initially
published from a local amd64 build and is now maintained by CI for both platforms.
Publishing uses the repository's `GITHUB_TOKEN` with
`packages: write`; no personal registry secret is required. A newly created GHCR
package defaults to private. Grant consumers read access, or change visibility
in the package settings if anonymous pulls are desired.

For the next PHP update, merge or cherry-pick the reviewed upstream source change,
update the full patch versions in both Trixie workflow matrices or the Alpine
workflow's `PHP_VERSION` environment variable and this table, and run
the workflow. Builds retain upstream's default PHP extensions and FPM settings.
Application-specific extensions and configuration belong in downstream images.

Example:

```dockerfile
FROM ghcr.io/palliis/php:8.3.36-fpm-trixie
```
