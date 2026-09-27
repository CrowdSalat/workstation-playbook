---
name: container-publish
description: "Use when publishing a container image to ghcr.io from GitHub Actions, adding or changing a multi-arch publish workflow, or deciding image tag and release semantics. Trigger keywords: publish image, ghcr, GitHub Actions, workflow, multi-arch, manifest list, buildx, build-push-action, metadata-action, image tag, release image."
---

# Container publish

Publish a `Containerfile`-built image to ghcr.io from GitHub Actions, multi-arch.

Not for authoring the `Containerfile` or local podman builds — see the `containers` skill.

## Policy
- Registry: `ghcr.io/<owner>/<repo>/<name>:<tag>`.
- File is `Containerfile`; workflows must pass `file: Containerfile` explicitly.
- Multi-arch (amd64+arm64) so cluster and ARM64 Mac pull without rebuild.

## Tags by trigger

| Trigger | Build | Manifest job | Tags published |
|---------|-------|--------------|-----------------|
| push to `main` | amd64 + arm64 lanes | yes | `:latest`, commit SHA |
| push tag `v*` | amd64 + arm64 lanes | yes | semver (`v1.2.3` → `1.2.3`), commit SHA, `:latest` |
| pull request to `main` | amd64 + arm64 lanes, no push, no cache write | no | none |

## Hard constraints
- Native runners only — `ubuntu-24.04` (amd64), `ubuntu-24.04-arm` (arm64). No
  `setup-qemu-action`, no `platforms: linux/amd64,linux/arm64` in one job.
- Each matrix lane pushes `<sha>-<arch>`; a separate `manifest` job merges them with
  `docker buildx imagetools create`. Lanes never write the final tag themselves.
- Image ref must be lowercase — compute per job with
  `echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY,,}/<name>" >> "$GITHUB_ENV"`. Actions has
  no `toLower()` function and `github.repository` keeps original case; GHCR rejects
  uppercase.
- Always set `file: Containerfile` on build-push-action — it defaults to `Dockerfile`
  and fails in Containerfile repos.
- Cache with `cache-from: type=gha` and `cache-to: type=gha,mode=max`; write the cache
  only on push, read it everywhere.
- Tag via `docker/metadata-action`: `type=raw,value=latest,enable={{is_default_branch}}`,
  `type=sha,prefix=` (bare short SHA), `type=semver,pattern={{version}}`.
- Keep action versions current with Dependabot: `.github/dependabot.yml` with
  `package-ecosystem: github-actions`, weekly interval.
- Add OCI labels `org.opencontainers.image.source` and `.revision` from `github.*`.

## Workflow

`.github/workflows/container.yml`:

```yaml
on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]
permissions:
  packages: write
jobs:
  build:
    runs-on: ${{ matrix.runner }}
    strategy:
      fail-fast: true
      matrix:
        include:
          - platform: linux/amd64
            runner: ubuntu-24.04
            arch: amd64
          - platform: linux/arm64
            runner: ubuntu-24.04-arm
            arch: arm64
    steps:
      - name: Set GHCR image name
        run: echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY,,}/<name>" >> "$GITHUB_ENV"
      - uses: actions/checkout@v7
      - uses: docker/setup-buildx-action@v4
      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - name: Build and push (native per-arch)
        uses: docker/build-push-action@v7
        with:
          context: .
          file: Containerfile
          platforms: ${{ matrix.platform }}
          push: ${{ github.event_name == 'push' }}
          tags: ${{ env.IMAGE }}:${{ github.sha }}-${{ matrix.arch }}
          labels: |
            org.opencontainers.image.source=${{ github.server_url }}/${{ github.repository }}
            org.opencontainers.image.revision=${{ github.sha }}
          cache-from: type=gha
          cache-to: ${{ github.event_name == 'push' && 'type=gha,mode=max' || '' }}

  manifest:
    needs: build
    if: github.event_name == 'push'
    runs-on: ubuntu-24.04
    steps:
      - name: Set GHCR image name
        run: echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY,,}/<name>" >> "$GITHUB_ENV"
      - uses: docker/setup-buildx-action@v4
      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/metadata-action@v6
        id: meta
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha,prefix=
            type=semver,pattern={{version}}
      - name: Create multi-arch manifests
        run: |
          while IFS= read -r tag; do
            [[ -z "$tag" ]] && continue
            docker buildx imagetools create \
              --tag "$tag" \
              "${{ env.IMAGE }}:${{ github.sha }}-amd64" \
              "${{ env.IMAGE }}:${{ github.sha }}-arm64"
          done <<< "${{ steps.meta.outputs.tags }}"
```
