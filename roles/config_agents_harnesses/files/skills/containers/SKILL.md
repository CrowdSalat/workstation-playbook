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
Publish by trigger:
- tag `v*` → semver release image
- `main` push → commit-sha image
- latest build → also tagged `:latest`

Images are multi-arch (amd64+arm64) — cluster and ARM64 Mac pull without rebuild; use native runners.

`.github/workflows/container.yml`:

```yaml
on:
  push:
    branches: [main]
    tags: ['v*']
permissions:
  packages: write
env:
  IMAGE: ghcr.io/${{ github.repository }}/<name>
jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-qemu-action@v3
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/metadata-action@v5
        id: meta
        with:
          images: ${{ env.IMAGE }}
          tags: |
            type=semver,pattern={{version}}
            type=sha
      - uses: docker/build-push-action@v6
        with:
          platforms: linux/amd64,linux/arm64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
```

## Image guidelines (OpenShift restricted-v3 SCC, default)
`restricted-v3` maps every UID to an unprivileged host UID; lower SCC only with explicit justification.

- No hardcoded UIDs — OpenShift assigns the runtime UID; never `chown` to fixed UID.
- Always GID 0 — writable dirs: `chmod 775 <dir> && chgrp 0 <dir>`.
- `USER 65534:0` — non-root default, portable to lower SCCs and PSA restricted.
- No host-root privileges — no `SYS_ADMIN`, `hostNetwork`, unmasked `/proc`, setuid/setgid reliance.
- Lean & secret-free — multi-stage + `.dockerignore`; exec-form `ENTRYPOINT`/`CMD` (PID 1); no secrets in `LABEL`/`ENV`.