# VSCodium Role

Installs and configures VSCodium with Microsoft VS Marketplace support.

## What This Role Does

1. **Installs VSCodium**
   - Linux: via Flatpak
   - macOS: via Homebrew cask

2. **Enables Microsoft VS Marketplace**
   - Configures `product.json` to use Microsoft's extension marketplace
   - Allows installing extensions directly from the VSCodium UI
   - No more manual VSIX downloads or Open VSX limitations

3. **Manages Settings** (optional)
   - Copies `files/settings.json` to VSCodium config directory
   - Controlled by `tool_vscodium_manage_settings` (default: true)
   - **Warning**: Will overwrite manual changes when enabled

4. **Installs Extensions**
   - Extensions listed in `tool_vscodium_extensions`
   - Simple format: just use the extension ID from marketplace
   - Example: `jetbrains.kotlin`, `ms-python.python`

5. **Adds Shell Aliases**
   - `code` and `codium` commands to launch VSCodium from terminal

## Configuration

### Variables (in `defaults/main.yml`)

```yaml
# Manage settings.json via Ansible (will overwrite manual changes)
tool_vscodium_manage_settings: true

# Extensions to install from Microsoft VS Marketplace
tool_vscodium_extensions:
  - stuart.unique-window-colors
  - jetbrains.kotlin
  - timheuer.pfx-view
```

### Settings File

Edit `files/settings.json` to customize VSCodium settings. Changes will be applied on next playbook run (if `tool_vscodium_manage_settings: true`).

## Extension Management

**Recommended workflow:**

1. **Install extensions via UI** - Browse Microsoft Marketplace in VSCodium and install
2. **Pin important ones in Ansible** - Add to `tool_vscodium_extensions` if you want them reinstalled automatically
3. **Keep the list minimal** - Only automate extensions you truly need on every setup

The Ansible extension list ensures baseline extensions are present, but you can freely install/remove others via the UI.

## Platform-Specific Behavior

### Linux (Flatpak)
- Config: `~/.config/VSCodium/User/`
- Product.json: `~/.var/app/com.vscodium.codium/config/VSCodium/product.json`

### macOS (Homebrew)
- Config: `~/.config/VSCodium/User/`
- Product.json: `/Applications/VSCodium.app/Contents/Resources/app/product.json`

## Changes from Previous Version

- **Removed complex extension installation** - No more marketplace/url mapping formats
- **Simplified to extension IDs only** - Just use the ID from VS Marketplace
- **Moved constants to tasks** - Only user-relevant variables in defaults
- **Settings via file copy** - No more JSON merging complexity
- **Enabled Microsoft Marketplace** - Install any extension via UI or Ansible
