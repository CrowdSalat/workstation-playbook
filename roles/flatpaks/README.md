# Role: flatpaks

Manages per-user Flatpak remotes and apps (`--user`, no sudo).

## What it does

- Ensures Flathub remote exists for the current user.
- Installs apps listed in `flatpak_packages`.
- Can also remove apps listed in `flatpak_packages` using a separate tag.

## Key variables

- `flatpak_packages`: list of Flatpak app IDs (for example, `com.vscodium.codium`).

## Tags

- `flatpaks`: install listed apps
- `flatpaks_remove`: remove listed apps

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags flatpaks
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags flatpaks_remove
```
