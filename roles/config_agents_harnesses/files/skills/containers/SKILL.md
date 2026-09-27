---
name: containers
description: Use when building container images locally with podman, authoring Containerfiles for OpenShift restricted-v3 SCC, or building and publishing multi-arch images to ghcr.io from GitHub Actions. Trigger keywords: container, Containerfile, Dockerfile, podman build, image build/push, ghcr, publish image, multi-arch.
---

# Containers

## Policy
- Podman, not Docker (unless the project requires Docker).
- Registry: `ghcr.io/<owner>/<repo>/<name>:<tag>`.
- File is `Containerfile`, never `Dockerfile`; pass `-f Containerfile`.

## Local build (fast, single-arch)
- `podman build -t localhost/<name>:<tag> .` — host platform only, no emulation.
- Push only for quick tests; normally not needed.
- Before tagging an existing release: `podman manifest inspect docker://ghcr.io/<owner>/<repo>/<name>:<tag>`.

## CI publish (GH Actions → ghcr, multi-arch)
Images are multi-arch (amd64+arm64) so cluster and ARM64 Mac pull without rebuild.

Publish by trigger:
- tag `v*` → semver image
- `main` push → commit-sha image
- default-branch build → also tagged `:latest`
- pull request to `main` → build only, no push, no cache write

Hard constraints:
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

## Image guidelines (OpenShift restricted-v3 SCC, default)
`restricted-v3` maps every UID to an unprivileged host UID; lower SCC only with explicit justification.

- No hardcoded UIDs — OpenShift assigns the runtime UID; never `chown` to fixed UID.
- Always GID 0 — writable dirs: `chmod 775 <dir> && chgrp 0 <dir>`.
- `USER 65534:0` — non-root default, portable to lower SCCs and PSA restricted.
- No host-root privileges — no `SYS_ADMIN`, `hostNetwork`, unmasked `/proc`, setuid/setgid reliance.
- Lean & secret-free — multi-stage + `.dockerignore`; exec-form `ENTRYPOINT`/`CMD` (PID 1); no secrets in `LABEL`/`ENV`.