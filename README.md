# Fedora Silverblue workstation playbook

Automates setup on Fedora Silverblue: per-user Flatpaks (under `~/.local/share/flatpak`, no sudo), CLI tools under `~/.local/bin`, shell/editor configuration, and optionally system packages via `rpm-ostree`.

Run Ansible on the host, not inside Toolbox. Flatpak and `rpm-ostree` tasks need host services.

## Quickstart

```bash
python3 -m venv .venv/ansible
.venv/ansible/bin/pip install ansible
.venv/ansible/bin/ansible-galaxy collection install community.general
cp vars/local.yml.example vars/local.yml
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml
```

## Prerequisites on the host

- `python3` (to create the venv)
- `jq` (used when merging VSCodium `settings.json`)
- Ansible collection `community.general` (Flatpak modules)

## Install Ansible in a project venv

From this directory:

```bash
python3 -m venv .venv/ansible
.venv/ansible/bin/pip install ansible
.venv/ansible/bin/ansible-galaxy collection install community.general
```

Use the venv's `ansible-playbook` path so you do not rely on system-wide Ansible.

Optional:

```bash
source .venv/ansible/bin/activate
```

## Personal vars (git identity, SSH signing key)

Copy the example file and fill it in. It is git-ignored and should never be committed.

```bash
cp vars/local.yml.example vars/local.yml
# edit vars/local.yml with your name, email, and signing key path
```

## Running playbooks

There are two playbooks:

| Playbook | sudo? | When to use |
|---|---|---|
| `user.yml` | No | Everyday routine for user-level roles |
| `system.yml` | Yes (`-K`) | System package layering (`rpm-ostree` + COPR) |

Run all user-level roles:

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml
```

Run system-level roles (requires reboot afterward):

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml system.yml -K
```

Run specific roles by tag:

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags tag1,tag2
```

## Roles

Detailed documentation now lives in each role directory:

| Role | Tag(s) | Docs |
|---|---|---|
| `bootstrap` | `bootstrap` | [`roles/bootstrap/README.md`](roles/bootstrap/README.md) |
| `ssh_agent` | `ssh_agent` | [`roles/ssh_agent/README.md`](roles/ssh_agent/README.md) |
| `starship` | `starship` | [`roles/starship/README.md`](roles/starship/README.md) |
| `flatpaks` | `flatpaks`, `flatpaks_remove` | [`roles/flatpaks/README.md`](roles/flatpaks/README.md) |
| `gnome_shell_extensions` | `gnome_shell_extensions` | [`roles/gnome_shell_extensions/README.md`](roles/gnome_shell_extensions/README.md) |
| `gnome_settings` | `gnome_settings` | [`roles/gnome_settings/README.md`](roles/gnome_settings/README.md) |
| `vscodium_config` | `vscodium_config` | [`roles/vscodium_config/README.md`](roles/vscodium_config/README.md) |
| `cursor` | `cursor` | [`roles/cursor/README.md`](roles/cursor/README.md) |
| `cli_tools` | `cli_tools` | [`roles/cli_tools/README.md`](roles/cli_tools/README.md) |
| `toolbox` | `toolbox` | [`roles/toolbox/README.md`](roles/toolbox/README.md) |
| `git` | `git` | [`roles/git/README.md`](roles/git/README.md) |
| `tool_configs` | `tool_configs` | [`roles/tool_configs/README.md`](roles/tool_configs/README.md) |
| `rpm_ostree` (`system.yml` only) | `rpm_ostree` | [`roles/rpm_ostree/README.md`](roles/rpm_ostree/README.md) |

Shared paths and version pins live in `vars/main.yml` and role `defaults/main.yml` files where applicable.
