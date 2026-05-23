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
