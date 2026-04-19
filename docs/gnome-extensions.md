# GNOME Extensions (later)

Saved notes for implementing workstation GNOME extension setup in the playbook.

## Checklist

- [x] Install Extension Manager (Flatpak: `com.mattjakeman.ExtensionManager`) — role `flatpaks` (per-user)
- [ ] Install and enable: Dash to Dock
- [ ] Install and enable: Night Theme Switcher
- [ ] Install and enable: System Monitor
- [ ] Install and enable: Places Status Indicator
- [ ] Install and enable: Applications Menu

## Extension Manager (Flatpak)

Install via the `flatpaks` role (same list as `flatpak_packages`); **per-user** Flatpak install. Run Ansible **on the host**, not inside Toolbox.

## Installing and enabling extensions from Ansible

GNOME Shell extensions are not Flatpaks. They are zip bundles installed under `~/.local/share/gnome-shell/extensions/<uuid>/` and listed in the `enabled-extensions` key.

### Constraints

1. **Shell version** — Each extension supports a range of GNOME Shell versions. Fedora Silverblue tracks current GNOME; verify each extension on [extensions.gnome.org](https://extensions.gnome.org) before pinning automation.
2. **Run on the host** — `gsettings` / `dconf` for the logged-in user need the correct session (DBUS). Running the playbook over SSH to `localhost` as the same user usually works; running only in a non-graphical context can be flaky for some keys.
3. **Order** — Install (unpack) first, then enable. Enabling an extension that is not installed fails.

### Practical approaches

| Approach | Pros | Cons |
|----------|------|------|
| **A. Extension Manager only** | Zero fragile automation; user picks versions | Not idempotent in Ansible beyond installing the Flatpak |
| **B. `gnome-extensions install` + enable** | Scriptable if you have zip URLs or local files | Need correct zip per Shell version; URLs on EGO change |
| **C. Download from EGO + unzip to extension dir** | Full control | Must resolve extension UUID and compatible version (API or fixed version pins) |
| **D. `dconf` / `gsettings` only** | Simple toggle | Only works **after** the extension files exist on disk |

Recommended path for automation: **C or B** for install, then **D** to append UUIDs to `org.gnome.shell enabled-extensions` (merge with existing list — do not overwrite unrelated extensions).

### Enable key (merge, do not replace)

- Schema: `org.gnome.shell`
- Key: `enabled-extensions` (string array of extension UUIDs)

Example (illustrative — confirm UUIDs on EGO):

```bash
gsettings get org.gnome.shell enabled-extensions
# Merge new UUIDs into the array; order can matter for some setups.
```

Ansible: `community.general.dconf` or `ansible.builtin.command` with `gsettings`. Prefer reading current value, merging in Ansible (`set_fact` + list), then writing back.

### UUIDs to look up (verify before automating)

Look up each extension on extensions.gnome.org and copy the **exact** UUID string (often `something@author`):

- Dash to Dock — common UUID: `dash-to-dock@micxgx.gnome.github.com` (confirm for your Shell)
- Night Theme Switcher — search EGO for the exact name
- System Monitor — often packaged differently; EGO name “System Monitor”
- Places Status Indicator — `places-status-indicator@gnome-shell-extensions.gcampax.github.com` (verify)
- Applications Menu — classic menu; verify UUID on EGO

Do not hardcode UUIDs in the playbook without verifying they match the extension versions you install.

## Suggested role layout (when you implement)

1. **`gnome_extensions` role** (or fold into `flatpaks` + one `gnome` role):
   - Vars: list of `{ name, zip_url_or_path, uuid }` or version pins per extension
   - Tasks: ensure Extension Manager Flatpak; ensure extension dirs; merge `enabled-extensions`
2. **Tags:** e.g. `gnome`, `extensions` for `--tags` runs
3. **Idempotency:** Check if `~/.local/share/gnome-shell/extensions/<uuid>/metadata.json` exists before download; compare enabled list before writing `gsettings`

## References

- [extensions.gnome.org](https://extensions.gnome.org)
- `man gnome-extensions` (on host with GNOME)
- Ansible: `community.general.dconf` module
