# Global Agent Rules

This file defines global guidance for AI coding agents (Claude Code, Pi, Cursor, etc.).
These rules apply across all projects unless overridden by project-specific AGENTS.md files.

## Git Commits

When writing commit messages (including suggested messages in chat):

- **[Conventional Commits](https://www.conventionalcommits.org/)** — use `type(optional scope): summary`. Pick a fitting `type` (`feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`, etc.); add a short `scope` when it clarifies the area (role name, component).
- **Subject** — imperative after the colon; the header line is at most **72 characters** total (including `type(scope): `), no trailing period.
- **Body** — Optional. Separate from the subject with a single blank line. Include a body only when it provides essential context for future reference (e.g., why a change was made, underlying bug root causes, or migration hints). Adopt the Linux kernel approach: use the body to explain the reasoning and intent behind the change, not to restate the code diff. Keep it tight, impactful, and free of conversational noise.
- **Footers** — none, except a **`BREAKING CHANGE:`** footer when documenting a breaking change, as Conventional Commits allows. Mark breaking commits with `!` on the type or scope when that is enough (`feat!:`, `feat(scope)!:`). **Never add `Co-Authored-By:` or other attribution footers** — git history tracks authorship automatically.
- **One logical change per commit** — do not mix unrelated topics; avoid splitting one logical change into many tiny commits.

Examples: `feat(teddycloud): add OCP entrypoint and Dockerfile`, `fix(metallb): correct BGP peer configuration`.

## Container Tools

- **Prefer Podman over Docker** — Use `podman` commands instead of `docker` unless Docker is explicitly required by the project.
- When suggesting containerization, default to Podman-compatible solutions.

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
