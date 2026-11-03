---
name: tool-installation
description: "Use when installing, updating, or adding tools to a workstation, or running this Ansible playbook. Trigger keywords: install, brew, dnf, flatpak, sdkman, mise, mise.toml, pipx, toolbox, apk, apt, ansible-playbook, tool role, new tool."
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

## Pick the channel first
Run before writing any task:

```bash
mise registry | grep -E '^<tool>[[:space:]]'   # a hit means mise can own the binary
```

| Result | Do this |
| --- | --- |
| Registry hit | add `<tool> = "latest"` to `roles/tool_developer/files/config.toml`; stop |
| Platform channel already owns it (flatpak, brew, Fedora RPM channel, pipx) | keep it in the existing `tool_<name>` role |
| Neither | write a full `tool_<name>` role |

Registry hit **plus** playbook-owned config (settings file, completions, extensions, harness detector dir) = both halves:
- `[tools]` entry in `roles/tool_developer/files/config.toml`
- `roles/tool_developer/tasks/postinstall-<tool>.yml`, wired from `tasks/main.yml`

Need two versions of one tool? Use a list — mise installs all, first is active:
```toml
java = ["temurin-25", "temurin-21"]
```

Never create a `tool_<name>` role whose only job is a mise entry.

Existing roles keep their channel — folding one into mise is its own commit.

## Add a global tool
Prefer extending an existing `tool_<name>`; create a new role only when nothing fits.

1. `tasks/main.yml` — install/postinstall phases; split by OS only when branching dominates.
2. Follow `ROLE_CONVENTIONS.md` (channel selection, phase layout, package manager helpers, CLI completion).
3. Register: Fedora `user-fedora.yml` (`roles_host`, or `roles_toolbox` for toolbox tools); macOS `user-macos.yml` (`roles_host`).
4. Fedora has two RPM channels: use `fedora_package_manager` (`rpm_ostree` on Atomic, `dnf` on classic) instead of hardcoding one. Atomic uses `roles_toolbox`; classic runs CLI tools on the host.
5. Toolbox context also needs `toolbox_name`; toolbox prereqs run only when `roles_toolbox | length > 0`. The playbook sets these per variant.

## Role families
- `tool_*`: user-facing intent (install + config)
- `package_manager_*`: install mechanics + manager readiness
- `config_*`: cross-cutting config

`mise` is not a `package_manager_*` role — it lives in `tool_developer` with the `[tools]` table it writes to.
`package_manager_*` stay toolbox-agnostic; the playbook controls toolbox delegation.

## Run the playbook
```bash
ansible-playbook -i inventory.yml user-fedora.yml                      # Fedora (auto-detects Atomic vs classic)
ansible-playbook -i inventory.yml user-fedora.yml -e fedora_variant=atomic   # force Atomic
ansible-playbook -i inventory.yml user-fedora.yml -e fedora_variant=classic  # force classic
ansible-playbook -i inventory.yml user-macos.yml                       # macOS
ansible-playbook -i inventory.yml user-fedora.yml -e '{"roles_host":["tool_vscodium"],"roles_toolbox":[]}'  # single role
```

Prereq: `pipx install --include-deps ansible`.