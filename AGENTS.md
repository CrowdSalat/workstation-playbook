# Agent instructions

Ansible playbooks and roles for **Fedora Silverblue** (immutable desktop, toolbox workflows). Treat guidance here as defaults; prefer what existing roles already do when they differ.

## Environment and assumptions

- Target users run **Silverblue** and use **toolboxes**; assume a **shared home directory** between the host and toolboxes when reasoning about paths, idempotency, and “where this file lands.”
- Prefer patterns that work without layering packages onto the host image unless there is no alternative.

## Playbook conventions

When adding or editing tasks in this repository:

- **GUI applications:** install with **Flatpak** using Ansible’s `community.general.flatpak` module (not ad-hoc `flatpak` shell unless the module cannot express the case).
- **CLI tools:** prefer shipping binaries or archives into **`~/.local/bin`** (user-writable, toolbox-friendly) rather than system paths unless the role already uses another layout.
- **Shell configuration:** add interactive-shell snippets to **`~/.bashrc`** for the managed user unless a role explicitly targets another shell.
- **Podman / Docker volumes:** when mounting host paths into containers, append the **`:Z`** SELinux relabel flag unless the playbook already handles labeling another documented way.
- **`rpm-ostree`:** do **not** introduce `rpm-ostree` tasks or advise host layering unless the user explicitly asks for it.

## Git commits

When writing commit messages (including suggested messages in chat):

- **[Conventional Commits](https://www.conventionalcommits.org/)** — use `type(optional scope): summary`. Pick a fitting `type` (`feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`, etc.); add a short `scope` when it clarifies the area (role name, component).
- **Subject** — imperative after the colon; the header line is at most **72 characters** total (including `type(scope): `), no trailing period.
- **Body** — optional. Add a blank line and a short body only when it helps (e.g. why, migration hint). Keep it tight; avoid long bodies that restate the diff.
- **Footers** — none, except a **`BREAKING CHANGE:`** footer when documenting a breaking change, as Conventional Commits allows. Mark breaking commits with `!` on the type or scope when that is enough (`feat!:`, `feat(scope)!:`).
- **One logical change per commit** — do not mix unrelated topics; avoid splitting one logical change into many tiny commits.

Examples: `feat(teddycloud): add OCP entrypoint and Dockerfile`, `fix(metallb): correct BGP peer configuration`.
