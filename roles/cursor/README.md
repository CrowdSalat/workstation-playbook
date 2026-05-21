# Role: cursor

Installs Cursor as an AppImage in the user profile.

## What it does

- Downloads Cursor AppImage into `~/Applications`
- Marks the AppImage executable
- Creates a desktop entry in `~/.local/share/applications`

## Tag

- `cursor`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags cursor
```
