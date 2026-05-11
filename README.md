# Fedora Silverblue workstation playbook

Automates setup on Fedora Silverblue: **per-user** Flatpaks (under `~/.local/share/flatpak`, no sudo), CLI tools under `~/.local/bin`, shell/editor configuration, and optionally system packages via `rpm-ostree`. Run Ansible **on the host**, not inside Toolbox — Flatpak and rpm-ostree tasks need the host daemon.

---

## Prerequisites on the host

- **Python 3** (to create the venv)
- `**jq`** — used when merging VSCodium `settings.json` (`dnf`/`rpm-ostree` in toolbox, or host if you layer it)
- `**community.general**` — Ansible collection for the Flatpak modules

---

## Install Ansible in a project venv

From this directory:

```bash
python3 -m venv .venv/ansible
.venv/ansible/bin/pip install ansible
.venv/ansible/bin/ansible-galaxy collection install community.general
```

Use the venv's `ansible-playbook` (path below) so you do not rely on a system-wide Ansible install.

Optional: `source .venv/ansible/bin/activate` and then call `ansible-playbook` without the full path.

---

## Personal vars (git identity, SSH signing key)

Copy the example file and fill it in — it is git-ignored and never committed:

```bash
cp vars/local.yml.example vars/local.yml
# edit vars/local.yml with your name, email, and signing key path
```

---

## Running the playbook

There are two playbooks:

| Playbook | sudo? | When to use |
|---|---|---|
| `user.yml` | **Never** | Everyday / routine — all user-level roles |
| `system.yml` | **Yes (`-K`)** | Adding/changing system packages (rpm-ostree + COPR) |

**Run all user-level roles:**

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml
```

**Run system-level roles** (COPR repos + rpm-ostree, requires reboot after):

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml system.yml -K
```

Pass `**--tags tag1,tag2**` (comma-separated, no spaces) to run only specific roles.

---

## What each role does

### bootstrap

Creates common directories: `~/.local/bin`, `~/.local/share/applications`, and `~/Applications`.

**Tag:** `bootstrap`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags bootstrap
```

### starship

Downloads the Starship binary into `~/.local/bin` and adds its init block to `~/.bashrc`.

**Tag:** `starship`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags starship
```

### flatpaks

Ensures the **Flathub** remote exists for **your user** (`flatpak remote-add --user`), then installs or removes Flatpaks with `--user` under `~/.local/share/flatpak` (no sudo).

Variables in `roles/flatpaks/defaults/main.yml`:

- `**flatpak_packages**` — list of Flatpak **app IDs** to install or uninstall (strings, e.g. `com.vscodium.codium`).

Install and remove are **separate tags** inside the role (the flatpaks role is not tagged at play level, so `--tags flatpaks_remove` does not run install tasks):

**Tag:** `flatpaks` — install everything listed in `flatpak_packages`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags flatpaks
```

**Tag:** `flatpaks_remove` — remove everything listed in `flatpak_packages`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags flatpaks_remove
```

### gnome_shell_extensions

Ensures `~/.local/share/gnome-shell/extensions` exists for per-user extensions. Installing or enabling specific extensions is left to Extension Manager (from **flatpaks**) or future automation.

**Tag:** `gnome_shell_extensions`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags gnome_shell_extensions
```

### vscodium_config

Configures VSCodium `settings.json` (integrated terminal via `flatpak-spawn` into your toolbox), installs extensions from **`vscodium_extensions`** (`roles/vscodium_config/defaults/main.yml`, needs network for remote VSIX), and adds `code` / `codium` shell functions (forward arguments to Flatpak) with **`blockinfile`**. It does **not** install the VSCodium Flatpak; the **flatpaks** role does.

**`vscodium_extensions`** entries can be:

- **Plain string** — `publisher.extension` via Open VSX / Codium default registry (`--install-extension`).
- **`url: https://…/file.vsix`** — download VSIX, then install from disk (cache: **`vscodium_vsix_cache_dir`**).
- **`marketplace:`** — `publisher`, `extension` (short id), `version` (from the [Visual Studio Marketplace](https://marketplace.visualstudio.com/)); downloads the official gallery VSIX then installs it.

**Tag:** `vscodium_config`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags vscodium_config
```

### cursor

Downloads the Cursor AppImage into `~/Applications`, makes it executable, and adds a `.desktop` file under `~/.local/share/applications`.

**Tag:** `cursor`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags cursor
```

### cli_tools

Installs CLI tools into `~/.local/bin`. Each entry in `cli_tools` (`roles/cli_tools/defaults/main.yml`) is either a direct binary download or a `.tar.gz` archive:

```yaml
cli_tools:
  # Direct binary — downloaded straight to ~/.local/bin/<name>
  - name: docker-compose
    url: "https://github.com/docker/compose/releases/download/v5.1.0/docker-compose-linux-x86_64"

  # Archive — downloaded, extracted, binary copied to ~/.local/bin/<name>
  - name: infisical
    url: "https://github.com/Infisical/cli/releases/download/v0.43.72/cli_0.43.72_linux_amd64.tar.gz"
    archive_binary: infisical   # filename inside the archive
```

**Tag:** `cli_tools`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags cli_tools
```

### git

Sets global `user.name`, `user.email`, and **SSH commit signing** (`gpg.format ssh` + `user.signingkey`). Values come from `vars/local.yml` (git-ignored); see `vars/local.yml.example` for the keys to set.

**Tag:** `git`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags git
```

### rpm_ostree *(system.yml only)*

Enables COPR repositories and layers packages into the Silverblue base image via `rpm-ostree`. **Requires sudo (`-K`) and a reboot to take effect.**

Variables in `roles/rpm_ostree/defaults/main.yml`:

- `**rpm_ostree_packages**` — list of `{ name, copr }` maps (omit `copr` if the package is in the default Fedora repos).

**Tag:** `rpm_ostree`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml system.yml -K --tags rpm_ostree
```

After the run, reboot to apply staged changes:

```bash
systemctl reboot
```

---

Shared paths and version pins live in `vars/main.yml` and in each role's `defaults/main.yml` where applicable.
