# Custom multi-arch smartctl-exporter image

Upstream `quay.io/prometheuscommunity/smartctl-exporter` ships **amd64 only**, so
it can't run on the arm64 Raspberry Pis. This repackages upstream's official
per-arch binary + `smartmontools` into a multi-arch image so one tag covers every
node.

## Build (CI — preferred)

`.github/workflows/smartctl-exporter-image.yml` builds and pushes this image to
`ghcr.io/mrzelee/smartctl-exporter` using the built-in `GITHUB_TOKEN` (no PAT).
It runs on pushes that touch this directory, or manually via **Actions → Build
smartctl-exporter image → Run workflow** (set the upstream version). The run
summary prints the `@sha256:` digest to pin in `../values.yaml`.

First time only: make the new GHCR package **public** (GitHub → your packages →
`smartctl-exporter` → Package settings → visibility → Public) so the cluster
pulls without an imagePullSecret.

## Build & push manually (fallback)

Needs `docker buildx` and a login to the target registry. Default target is
GitHub Container Registry under your account (public → no pull secret needed):

```sh
# one-time: login (use a PAT with write:packages)
echo "$GHCR_PAT" | docker login ghcr.io -u MrZeLee --password-stdin

# build both arches and push
docker buildx build --platform linux/amd64,linux/arm64 \
  --build-arg VERSION=0.14.0 \
  -t ghcr.io/mrzelee/smartctl-exporter:v0.14.0 \
  --push \
  09-monitoring/smartctl-exporter/image
```

Then make the package **public** once (GitHub → Packages → smartctl-exporter →
Package settings → Change visibility → Public), so the cluster can pull it
without an imagePullSecret.

To use a different registry (e.g. the in-cluster GitLab), change the `-t` tag
here **and** `image.repository`/`image.tag` in `../values.yaml` to match — and
add an imagePullSecret if that registry is private.

## Bumping the version

Change `VERSION` in both the build command and `../values.yaml` `image.tag`,
rebuild, push.
