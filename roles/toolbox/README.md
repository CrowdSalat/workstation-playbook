# Role: toolbox

Ensures a toolbox container exists and installs packages inside it.

## Key variables

- `toolbox_dnf_packages`: DNF packages installed inside toolbox
- `toolbox_npm_packages`: global npm packages installed inside toolbox
- `toolbox_pip_packages`: user pip packages installed inside toolbox
- `toolbox_sdkman_candidates`: SDKMAN candidates to install

SDKMAN is bootstrapped automatically when needed, and init is added to `~/.bashrc`.

## Tag

- `toolbox`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags toolbox
```
