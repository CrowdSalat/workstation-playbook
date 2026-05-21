# Role: bootstrap

Creates common user directories used by other roles:

- `~/.local/bin`
- `~/.local/share/applications`
- `~/Applications`

## Tag

- `bootstrap`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags bootstrap
```
