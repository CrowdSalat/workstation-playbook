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

## Lint
- `hadolint Containerfile` before committing a Containerfile change, if hadolint is available

## Image guidelines

### General
- Run as non-root — set an explicit `USER`; never ship an image that defaults to root.
- Lean — multi-stage builds + `.dockerignore`; copy only the built artifact out of the builder stage.
- Exec-form `ENTRYPOINT`/`CMD` — the binary must be PID 1 to receive signals.
- Secret-free — no credentials in `LABEL`, `ENV`, or build args; nothing sensitive baked into layers.
- Base image — `scratch` only for `CGO_ENABLED=0` static binaries; otherwise `alpine`, which keeps `sh` and a package manager.
- Cache mounts — `RUN --mount=type=cache,target=<dir>` reuses package downloads across rebuilds without adding image layers.
- Layer order — `COPY` manifests and lockfiles first, `RUN` the install, `COPY` the source last. Any source edit above the install busts the dependency layer.
- Combine dependent `RUN` steps and clean package indexes in the same layer (`apk` cache, `apt-get` lists). A later layer cannot shrink an earlier one.
- Put what changes most last; a change only invalidates layers from its point onward.

### OpenShift restricted-v3 SCC, default
`restricted-v3` maps every UID to an unprivileged host UID and always runs with GID 0; lower SCC only with explicit justification.

- No hardcoded UIDs — OpenShift assigns the runtime UID; never `chown` to fixed UID.
- Always GID 0 — writable dirs: `chmod 775 <dir> && chgrp 0 <dir>`, for every path a non-`scratch` base needs to write.
- `USER 65534:0` — the UID is a placeholder the platform overrides; GID 0 is what makes it portable to lower SCCs and PSA restricted.
- No host-root privileges — no `SYS_ADMIN`, `hostNetwork`, unmasked `/proc`, setuid/setgid reliance.
