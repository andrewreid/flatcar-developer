# Flatcar Container Linux Development Container

This repository automatically builds OCI container images of the Flatcar Container
Linux developer image, publishes them to the GitHub Container Registry (GHCR), and
cuts a GitHub Release for every Flatcar version it sees.

The developer image is a fuller Linux than the production Flatcar image — it ships
the GCC toolchain and the userspace needed to compile binaries, drivers, and
kernel modules that target Flatcar. Wrapping it in a Docker image lets you use it
inside the familiar `docker build` / multi-stage workflow, so you can produce
slim runtime images without dragging the build chain along.

## Image variants

Two tags are produced for every Flatcar version `<X.Y.Z>`:

| Tag | Contents |
|-----|----------|
| `<X.Y.Z>` | Base developer container, rebuilt from the root filesystem in the upstream `flatcar_developer_container.bin.bz2`. |
| `<X.Y.Z>-sources` | Base image extended with the matching kernel sources at `/usr/src/linux` and `coreos-overlay` / `portage-stable` checked out at the version's tag. Use this as a starting point for building out-of-tree kernel modules guaranteed to match the running kernel. |

Each track also has moving aliases (`stable`, `beta`, `alpha`, plus the `-sources`
variants) that always point at the latest published version for that track.

## Pulling images

Images are published to: <https://github.com/andrewreid/docker-flatcar-developer/pkgs/container/docker-flatcar-developer>

```
docker pull ghcr.io/andrewreid/docker-flatcar-developer:<X.Y.Z>
docker pull ghcr.io/andrewreid/docker-flatcar-developer:<X.Y.Z>-sources

# or follow a track
docker pull ghcr.io/andrewreid/docker-flatcar-developer:stable
docker pull ghcr.io/andrewreid/docker-flatcar-developer:stable-sources
```

Each Flatcar version also gets a [GitHub Release](https://github.com/andrewreid/docker-flatcar-developer/releases)
named after the version, with release notes listing the published image tags and
their manifest digests.

## Verifying image provenance

Each published image carries BuildKit provenance stored in GHCR alongside the
image. Verify provenance with the GitHub CLI:

```
gh attestation verify oci://ghcr.io/andrewreid/docker-flatcar-developer:<X.Y.Z> \
  --owner andrewreid
```

## How the build works

A single GitHub Actions workflow ([`.github/workflows/build.yaml`](.github/workflows/build.yaml))
runs daily and on `workflow_dispatch`. It fans out across the three Flatcar
release channels (`stable`, `beta`, `alpha`) as a matrix, and each leg runs end
to end independently:

1. Read `https://<track>.release.flatcar-linux.net/amd64-usr/current/version.txt`
   to discover the channel's current `FLATCAR_VERSION`.
2. Probe GHCR via `docker buildx imagetools inspect` to see whether `:<version>` and
   `:<version>-sources` already exist; skip whichever variants are already
   published (idempotent — daily cron is cheap).
3. For the base image: download `flatcar_developer_container.bin.bz2`, loop-mount
   the partition at offset `2097152`, extract `rootfs.tar`, and publish it via
   [Dockerfile.base](./Dockerfile.base) and `docker/build-push-action`.
4. For the sources image: build the [Dockerfile](./Dockerfile) on top of the
   freshly pushed base, with GHA buildx layer cache, and push.
5. Publish BuildKit provenance with each image push.
6. Create or update a GitHub Release named after the Flatcar version from a
   serialized release job, with body derived from the registry so matrix legs
   that happen to share a version converge on identical content.

Authentication uses `secrets.GHCR_TOKEN || secrets.GITHUB_TOKEN`. This public
fork still needs a classic PAT in `GHCR_TOKEN` for GHCR pushes; the built-in
`GITHUB_TOKEN` remains the fallback for non-fork deployments.

To force a rebuild of the current versions even when tags already exist, run the
workflow manually with `force=true`. To restrict to a single channel, set
`stream` to `stable`, `beta`, or `alpha`.

### Backfill

The workflow only ever builds the *current* release of each channel. If you need
to publish images for older Flatcar versions, do it manually — there is no
automated back-catalogue build.

## Credits

Originally created by [@AnalogJ](https://github.com/AnalogJ) at
[mediadepot/docker-flatcar-developer](https://github.com/mediadepot/docker-flatcar-developer),
which itself was inspired by [BugRoger/coreos-developer-docker](https://github.com/BugRoger/coreos-developer-docker).
The original repo published to Docker Hub under
`mediadepot/flatcar-developer`.

This fork keeps the same intent — turn the upstream Flatcar developer container
and its kernel-sources companion into a pair of Docker images — but rewrites the
delivery pipeline:

- **Registry**: Docker Hub → GitHub Container Registry.
- **Auth**: hardcoded Docker Hub credentials → `GHCR_TOKEN` with `GITHUB_TOKEN`
  fallback.
- **Releases**: none → one GitHub Release per Flatcar version, named after that
  version (e.g. `4593.2.0`), with image digests in the body.
- **Pipeline shape**: monolithic shell step → matrix-per-track job using
  `docker/login-action`, `docker/setup-buildx-action`, `docker/metadata-action`,
  and `docker/build-push-action` with GHA layer cache.
- **Idempotency**: ad-hoc Docker Hub tags-API scrape → `docker buildx imagetools inspect`
  against GHCR.
- **Other**: SHA-pinned actions with Dependabot refreshes, scoped `permissions:`,
  `concurrency:` group, daily cron, dropped `keepalive-workflow`.

## References

- [Flatcar — Developing for Flatcar with the SDK](https://www.flatcar.org/docs/latest/reference/developer-guides/sdk-modifying-flatcar/)
- [Flatcar — Building custom kernel modules](https://www.flatcar.org/docs/latest/setup/customization/kernel-modules/)
- [BugRoger/coreos-developer-docker](https://github.com/BugRoger/coreos-developer-docker) (original inspiration)
- [mediadepot/docker-flatcar-developer](https://github.com/mediadepot/docker-flatcar-developer) (upstream fork source)
