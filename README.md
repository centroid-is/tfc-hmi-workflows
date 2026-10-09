# tfc-hmi-workflows

CI, builds and releases for [centroid-is/tfc-hmi](https://github.com/centroid-is/tfc-hmi),
which is private. The code lives there; the workflows, the Actions minutes
and the published releases live here.

Every job starts by checking tfc-hmi out into `src/` with the `GH_PAT`
secret (`.github/actions/checkout-source`), so every path in a workflow is
`src/...`. Events arrive from one small workflow inside tfc-hmi,
[`bridge/tfc-hmi-dispatch.yml`](bridge/tfc-hmi-dispatch.yml). Pushes to main
and tags come as `repository_dispatch`. A pull request comes as a mirror pull
request: the bridge keeps a branch `tfc-hmi/pr-N` here whose one file,
`tfc-hmi-source.json`, names the commit to test, force-pushes it on every push
to the original, opens the mirror PR with no description, and closes it when
the original closes.

## What runs when

| Event in tfc-hmi | Here | Result |
|---|---|---|
| pull request opened or pushed | `Tests`, on the mirror PR `tfc-hmi#N` here | every test suite, every build, image validation; checks on the mirror PR, verdict posted back to the original as the commit status `tfc-hmi-workflows / tests` |
| push to `main` | `Tests`, `Main Prerelease` | the `main-latest` prerelease of this repository is republished with all assets; `:latest`, `:latest-release` and `:latest-profile` images pushed to ghcr.io |
| version tag `vYYYY.M.D` pushed | `Tag Release` | version written into `centroid-hmi/pubspec.yaml` and pushed to tfc-hmi main; a release with that tag is created here; `:stable` images pushed |
| by hand | any workflow, with a `ref` | same as above for that ref; `Station image` can also publish a `station-v*` release of the OS image |

Releases live in this repository, so the app's update channels and
centroidx-manager must point at `centroid-is/tfc-hmi-workflows`
(`centroid-hmi/lib/main.dart` and `tools/centroidx-manager/main.go`).

## Workflows

| File | Called by | Produces |
|---|---|---|
| `test.yml` | bridge, hand | the verdict; calls every builder below |
| `linux.yml` | test, main-prerelease | `centroidx_linux_x64.tar.gz` |
| `macos.yml` | test, main-prerelease, tag-release | `centroidx_darwin_arm64.dmg`, signed and notarized when the Apple secrets are set |
| `windows.yml` | test, main-prerelease, tag-release | `centroid-hmi.msix` (signed sideload package), `centroidx-sideload.cer`, the portable Release folder |
| `manager.yml` | test, main-prerelease, tag-release | `centroidx-manager_{windows_amd64.exe,linux_amd64,darwin_arm64.dmg}`, unit and integration tests |
| `elinux.yml` | test, main-prerelease, tag-release | ghcr.io images `centroid-hmi`, `centroid-backend`, `hmi-profiler` |
| `station-image.yml` | test (validate only), main-prerelease, tag-release, hand | the station OS image and the USB installer |
| `main-prerelease.yml` | bridge, hand | the `main-latest` release |
| `tag-release.yml` | bridge, hand | the stable release |

Shared pieces in `.github/actions`: `checkout-source`, `setup-flutter` (the
version pinned in tfc-hmi's `.flutter-version`), `pub-get` (retries only
pub.dev's transient failures), `docker-build-push` (one retry against ghcr.io
hiccups), `ghcr-login`. `.github/scripts/create-dmg.sh` retries `hdiutil`.

## Secrets

Set on this repository:

| Secret | Used for |
|---|---|
| `GH_PAT` | reading tfc-hmi and its private git dependencies (open62541_dart, postgresql-dart, board_datetime_picker, cristalyse); pushing the version bump to tfc-hmi main; posting commit statuses to tfc-hmi; pulling `tfc-toolchain` and `weston` and pushing the station images on ghcr.io |
| `MSIX_CERT_PFX_BASE64`, `MSIX_CERT_PASSWORD` | signing the manager and the MSIX with the Centroid sideload certificate |
| `APPLE_CERTIFICATE_P12_BASE64`, `APPLE_CERTIFICATE_PASSWORD`, `APPLE_TEAM_ID`, `APPLE_ID`, `APPLE_ID_PASSWORD` | signing and notarizing the macOS builds; unsigned builds when absent |

`GH_PAT` has to be a **classic** token with `repo`, `write:packages` and
`workflow`: fine-grained tokens cannot be granted package scopes, and the
images are owned by the organisation, not by this repository.

Set on tfc-hmi:

| Secret | Used for |
|---|---|
| `WORKFLOWS_DISPATCH_TOKEN` | the bridge workflow, which runs on a self-hosted runner (git, gh and bash; no Actions minutes): `repository_dispatch`, the mirror branches and the mirror pull requests here (Contents and Pull requests read/write on this repository) |

## Running things by hand

Every workflow has `workflow_dispatch` with a `ref` (tfc-hmi commit, branch
or tag). `Tag Release` takes the tag name instead. `Station image` takes
`gpu`, `channel`, `preload` and an optional `release_tag`.

```
gh workflow run test.yml -R centroid-is/tfc-hmi-workflows -f ref=my-branch
gh workflow run tag-release.yml -R centroid-is/tfc-hmi-workflows -f tag=v2026.10.9
```
