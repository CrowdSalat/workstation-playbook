# Global Agent Rules

This file defines global guidance for AI coding agents (Claude Code, Pi, Cursor, etc.).
These rules apply across all projects unless overridden by project-specific AGENTS.md files.

## Git Commits

Keep commits atomic: commit only the files you touched and list each path explicitly. For tracked files run `git commit -m "<scoped message>" -- path/to/file1 path/to/file2`. For brand-new files, use the one-liner `git restore --staged :/ && git add "path/to/file1" "path/to/file2" && git commit -m "<scoped message>" -- path/to/file1 path/to/file2`

When writing commit messages (including suggested messages in chat):

- **[Conventional Commits](https://www.conventionalcommits.org/)** — use `type(optional scope): summary`. Pick a fitting `type` (`feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`, etc.); add a short `scope` when it clarifies the area (role name, component).
- **Subject** — imperative after the colon; the header line is at most **72 characters** total (including `type(scope): `), no trailing period.
- **Body** — Optional. Separate from the subject with a single blank line. Include a body only when it provides essential context for future reference (e.g., why a change was made, underlying bug root causes, or migration hints). Adopt the Linux kernel approach: use the body to explain the reasoning and intent behind the change, not to restate the code diff. Keep it tight, impactful, and free of conversational noise. **Never hallucinate reasons or include placeholders (like `TODO` or `[Fix ID]`).**
- **Footers** — none, except a **`BREAKING CHANGE:`** footer when documenting a breaking change, as Conventional Commits allows. Mark breaking commits with `!` on the type or scope when that is enough (`feat!:`, `feat(scope)!:`). **Use standard issue closing keywords (e.g., `Closes #123` or `Fixes PROJ-456`) on a separate line at the bottom if an issue/ticket exists.** Never add `Co-Authored-By:` or other attribution footers — git history tracks authorship automatically.
- **No Agent Meta-Language** — Do not include conversational filler or references to your identity as an AI (e.g., avoid *"As requested,"* *"Per instructions,"* or *"I have fixed..."*). The commit must read as if written by a core maintainer.
- **One logical change per commit** — do not mix unrelated topics; avoid splitting one logical change into many tiny commits. **If a tool-assisted change affects many files (e.g., a global renaming or dependency bump), explicitly state the macro-reason in the body.**

Examples: 
- `feat(teddycloud): add OCP entrypoint and Dockerfile`
- `fix(metallb): correct BGP peer configuration`
- `docs(readme): document new environment variables`

## Container Tools

- **Prefer Podman over Docker** — Use `podman` commands instead of `docker` unless Docker is explicitly required by the project. Default to Podman-compatible solutions.
- **Registry** — Publish images to the repository's GitHub Container Registry: `ghcr.io/<owner>/<repo>/<name>:<tag>`.
- **Publish modes** (driven by GitHub Actions):
  - *Tag `v*` (semver):* versioned release image.
  - *`main` push:* commit-sha build.
  - *Latest build:* additionally tagged `:latest`.
- **Published images are multi-arch** (AMD64 + ARM64) so both the x86_64 cluster and the ARM64 Mac pull without rebuilding; use native runners in Actions to keep builds fast.
- **Local build is fast and single-arch** — target only the host platform, no emulation: `podman build -t localhost/<name>:<tag> .`. Local push is fine for quick tests but normally not needed; before tagging an existing release, check `podman manifest inspect docker://ghcr.io/<owner>/<repo>/<name>:<tag>`.

**Actions example** — multi-arch publish to GHCR:

```yaml
# .github/workflows/container.yml
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

## Container Image Guidelines

- **Use `Containerfile`, not `Dockerfile`** — Always name container build files `Containerfile` (OCI standard). Use `-f Containerfile` in build commands when needed.
- **OpenShift `restricted-v3` SCC compatibility (default)** — Target OpenShift's `restricted-v3` SCC by default; drop to a lower SCC (e.g. `restricted-v2`, `nonroot-v2`) only when there is a concrete reason, and justify it explicitly. `restricted-v3` runs pods in a Linux user namespace (`hostUsers: false`), so the container's UID (even 0) is mapped to an unprivileged host UID:
  - **No hardcoded UIDs** — OpenShift assigns a random UID at runtime. Never `chown` to a fixed UID.
  - **Use GID 0 (root group)** — OpenShift always assigns GID 0. Make writable directories group-accessible: `chmod 775 <dir> && chgrp 0 <dir>`.
  - **Set a non-root default USER** — Use `USER 65534:0` (`nobody` with root group) as the default. The UID is overridden by the runtime; a non-root `USER` keeps the image portable to `restricted-v2`, `restricted`, and plain Kubernetes (PSA restricted).
  - **Do not assume host-root privileges** — Inside the user namespace "root" is unprivileged on the host. No `SYS_ADMIN`, `hostNetwork`, unmasked `/proc`, or `setuid`/`setgid` binary reliance; flag any needed capability or privilege escalation explicitly.
  - **Keep images lean and secret-free** — Multi-stage builds with a `.dockerignore`, exec-form `ENTRYPOINT`/`CMD` (app as PID 1, receives signals), no secrets baked in or leaked via `LABEL`/`ENV`.

## Tool Installation Philosophy

- **Use Ansible playbook for tool installation** — Do not suggest ad-hoc installations via `curl | bash`, `brew install`, or manual downloads.
- New tools should be added as roles in the `workstation-playbook` repository.
- Follow the role architecture:
  - `tool_*`: user-facing tool intent (install + tool-specific config)
  - `package_manager_*`: installation mechanics and manager readiness
  - `config_*`: cross-cutting configuration not owned by one tool

## Code Quality & Safety

- **Idempotency** — Keep idempotency in mind for all tasks and scripts.
- **No destructive git operations** — Avoid `git reset --hard`, `git push --force` to main/master, or rewriting published history without explicit user confirmation.
- **Explicit over implicit** — Prefer explicit defaults and clear variable naming.
- **Security** — Never commit secrets, API keys, or credentials to version control.

## Editing Behavior

- Do not revert unrelated user changes.
- When making structural changes, validate syntax before committing.
- Prefer focused, incremental changes over large refactors.
