# Role: gnome_settings

Applies GNOME settings through `gsettings` in an idempotent way.

## Default behavior

Current defaults include Alt+Tab behavior tuned for window switching:

- clears `switch-applications`
- sets `switch-windows` to `<Alt>Tab`
- sets `switch-windows-backward` to `<Shift><Alt>Tab`

## Key variables

- `gnome_keybindings` (and related lists in `defaults/main.yml`)

Override in inventory or vars files as needed.

## Tag

- `gnome_settings`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags gnome_settings
```
