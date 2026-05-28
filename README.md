# workstation playbook

This repository contains the workstation automation playbooks and roles.

## Run on Fedora

```bash
ansible-playbook -i inventory.yml user-fedora.yml
```

## Run on macOS

```bash
ansible-playbook -i inventory.yml user-macos.yml
```

## Run single roles

Use playbook variable overrides to run only selected roles.

```bash
ansible-playbook -i inventory.yml user-fedora.yml -e '{"roles_host":["tool_vscodium"],"roles_toolbox":[]}'
```

## Install Ansible prerequisite

```bash
python3 -m pip install --user pipx
python3 -m pipx ensurepath
pipx install --include-deps ansible
```
