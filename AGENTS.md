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

When writing commit messages (including suggested messages in chat), follow this strictly:

- **Subject line only** — a single line; do not add a body or extra paragraphs unless the user explicitly asks for a longer message.
- **No footer block** — do not append trailers, `Signed-off-by:`, `Co-authored-by:`, `Reviewed-by:`, or any signature-style lines after the subject.
- **Subject rules** — imperative mood (e.g. "Add…", "Fix…"), at most 72 characters, no trailing period.
- **One logical change per commit** — do not mix unrelated topics; avoid splitting one logical change into many tiny commits.

Examples of acceptable subjects: `Add teddycloud OCP entrypoint and Dockerfile`, `Fix metallb BGP peer configuration`.

Do not propose or use multi-line commit templates with a blank line and body, and do not "helpfully" add footers every time.
