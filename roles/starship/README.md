# Role: starship

Installs the Starship prompt binary into `~/.local/bin` and ensures init is present in `~/.bashrc`.

## Tag

- `starship`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags starship
```
