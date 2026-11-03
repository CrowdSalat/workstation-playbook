# workstation playbook

This repository contains the workstation automation playbooks and roles.

## Run on Fedora

`user-fedora.yml` detects whether the host is Fedora Atomic (`/run/ostree-booted`)
or traditional Fedora and selects the RPM channel automatically:

- Atomic: `rpm-ostree`, with `tool_cli_utils` in the toolbox.
- Traditional: `dnf` on the host, without a toolbox layer.

```bash
ansible-playbook -i inventory.yml user-fedora.yml
```

Force a variant when detection is not appropriate:

```bash
ansible-playbook -i inventory.yml user-fedora.yml -e fedora_variant=atomic
ansible-playbook -i inventory.yml user-fedora.yml -e fedora_variant=classic
```

Add `--ask-become-pass` when `sudo` is not passwordless.

## Run on macOS

```bash
ansible-playbook -i inventory.yml user-macos.yml
```

## Run single roles

Use playbook variable overrides to run only selected roles.

```bash
ansible-playbook -i inventory.yml user-fedora.yml -e '{"roles_host":["tool_developer"],"roles_toolbox":[]}'
```

## Install Ansible prerequisite

```bash
python3 -m pip install --user pipx
python3 -m pipx ensurepath
pipx install --include-deps ansible
```
