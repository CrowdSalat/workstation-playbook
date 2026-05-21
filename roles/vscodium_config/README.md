# Role: vscodium_config

Configures VSCodium user settings and extensions for a Flatpak-based workflow.

## What it does

- Merges VSCodium `settings.json` (toolbox-friendly terminal setup).
- Installs extensions from `vscodium_extensions`.
- Adds `code` and `codium` shell functions that forward arguments to the Flatpak app.

This role does not install VSCodium itself; use the `flatpaks` role for that.

## Key variables

- `vscodium_extensions`
  - plain string: `publisher.extension`
  - `url`: direct VSIX URL
  - `marketplace`: `publisher`, `extension`, `version`
- `vscodium_vsix_cache_dir`: local VSIX cache path

## Tag

- `vscodium_config`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags vscodium_config
```
