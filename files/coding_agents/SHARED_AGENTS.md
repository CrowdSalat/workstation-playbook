# Global Agent Rules

This file defines global guidance for AI coding agents (Claude Code, Pi, Cursor, etc.).
These rules apply across all projects unless overridden by project-specific AGENTS.md files.

## Git Commits

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
- **Container Registry** — Always push images to `docker.io` (Docker Hub) as the default registry.
- **Multi-Architecture Builds** — Build images for both AMD64 (x86_64) and ARM64. AMD64 is the **primary target and must always be built**. ARM64 should be included whenever possible.
- **Multi-Arch Workflow** — Use `--manifest` (not `-t`) to build a proper manifest list, and `podman manifest push` (not `podman push`) to push it. Using `-t` with multiple platforms silently produces only a single-arch image.
  - *Build:* `podman build --platform linux/amd64,linux/arm64 --manifest docker.io/username/image:tag .`
  - *Fallback (If ARM64 fails/is blocked):* `podman build --platform linux/amd64 --manifest docker.io/username/image:tag .`
  - *Push:* `podman manifest push --all docker.io/username/image:tag docker://docker.io/username/image:tag`
  - *Cleanup (optional):* `podman manifest rm docker.io/username/image:tag`
- **Version Safety** — Before building and pushing, check whether the target tag already exists in the registry: `podman manifest inspect docker://docker.io/username/image:tag`. If it exists, do not overwrite it — bump to the next available version or ask the user which version to use.

## Container Image Guidelines

- **Use `Containerfile`, not `Dockerfile`** — Always name container build files `Containerfile` (OCI standard). Use `-f Containerfile` in build commands when needed.
- **OpenShift `restricted-v2` SCC compatibility** — All container images must run under OpenShift's `restricted-v2` Security Context Constraint by default:
  - **No hardcoded UIDs** — OpenShift assigns a random UID at runtime. Never `chown` to a fixed UID.
  - **Use GID 0 (root group)** — OpenShift always assigns GID 0. Make writable directories group-accessible: `chmod 775 <dir> && chgrp 0 <dir>`.
  - **Set a non-root default USER** — Use `USER 65534:0` (`nobody` with root group) as the default. OpenShift overrides the UID but keeps GID 0.
  - **Avoid privileged operations at runtime** — No host mounts, no `SYS_ADMIN`, no `hostNetwork` unless technically required. If a capability or privilege escalation is needed, flag it explicitly and explain why it cannot be avoided.

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
