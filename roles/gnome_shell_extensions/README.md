# Role: gnome_shell_extensions

Ensures the per-user GNOME Shell extension directory exists:

- `~/.local/share/gnome-shell/extensions`

This role does not install or enable specific extensions yet.

## Tag

- `gnome_shell_extensions`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags gnome_shell_extensions
```
