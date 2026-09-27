---
name: containers
description: "Use when authoring or fixing a Containerfile for OpenShift restricted-v3, or building container images locally with podman. Trigger keywords: container, Containerfile, Dockerfile, podman build, image build, restricted-v3, SCC, scratch, multi-stage."
---

# Containers

Author a `Containerfile` and build it locally.

Not for publishing to a registry from CI — see the `container-publish` skill.

## Policy
- Podman, not Docker (unless the project requires Docker).
- File is `Containerfile`, never `Dockerfile`; pass `-f Containerfile`.
- Registry images are `ghcr.io/<owner>/<repo>/<name>:<tag>`.

## Local build (fast, single-arch)
- `podman build -t localhost/<name>:<tag> .` — host platform only, no emulation.
- Push only for quick tests; normally not needed.
- Before tagging an existing release: `podman manifest inspect docker://ghcr.io/<owner>/<repo>/<name>:<tag>`.

## Image guidelines (OpenShift restricted-v3 SCC, default)
`restricted-v3` maps every UID to an unprivileged host UID; lower SCC only with explicit justification.

- No hardcoded UIDs — OpenShift assigns the runtime UID; never `chown` to fixed UID.
- Always GID 0 — writable dirs: `chmod 775 <dir> && chgrp 0 <dir>`.
- `USER 65534:0` — non-root default, portable to lower SCCs and PSA restricted.
- No host-root privileges — no `SYS_ADMIN`, `hostNetwork`, unmasked `/proc`, setuid/setgid reliance.
- Lean & secret-free — multi-stage + `.dockerignore`; exec-form `ENTRYPOINT`/`CMD` (PID 1); no secrets in `LABEL`/`ENV`.
- `scratch` base is fine for `CGO_ENABLED=0` static binaries only; otherwise `alpine` and `chmod 775` + `chgrp 0` every writable path.
