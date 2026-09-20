---
name: tool-installation
description: Use when installing, updating, or adding tools to a workstation, or running this Ansible playbook. Trigger keywords: install, brew, dnf, flatpak, sdkman, mise, mise.toml, pipx, toolbox, apk, apt, ansible-playbook, tool role, new tool.
---

# Installing tools

## Policy
- No silent ad-hoc global installs — `curl | bash`, `brew install`, manual downloads.
- Project-scoped tool → local `mise.toml` in the project.
- Globally useful tool → role in the playbook via `ansible-playbook`.

## Scope first
- Project-only: add to the project's `mise.toml` (respect existing file; mise = the workstation version manager):

  ```toml
  [tools]
  golang = "1.22"
  ```

- Globally useful: propose adding to an existing `tool_<name>` role (or a new one); on agreement add it and run the playbook in the same session.

## Add a global tool
Prefer extending an existing `tool_<name>`; create a new role only when nothing fits.

1. `tasks/main.yml` — install/postinstall phases; split by OS only when branching dominates.
2. Follow `ROLE_CONVENTIONS.md` (phase layout, package manager helpers, CLI completion).
3. Register: Fedora `user-fedora.yml` (`roles_host`, or `roles_toolbox` for toolbox tools); macOS `user-macos.yml` (`roles_host`).
4. Toolbox context also needs `toolbox_name`; toolbox prereqs run only when `roles_toolbox | length > 0`.

## Role families
- `tool_*`: user-facing intent (install + config)
- `package_manager_*`: install mechanics + manager readiness
- `config_*`: cross-cutting config

`package_manager_*` stay toolbox-agnostic; the playbook controls toolbox delegation.

## Run the playbook
```bash
ansible-playbook -i inventory.yml user-fedora.yml                      # Fedora
ansible-playbook -i inventory.yml user-macos.yml                       # macOS
ansible-playbook -i inventory.yml user-fedora.yml -e '{"roles_host":["tool_vscodium"],"roles_toolbox":[]}'  # single role
```

Prereq: `pipx install --include-deps ansible`.