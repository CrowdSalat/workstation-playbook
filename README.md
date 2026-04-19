# Fedora Silverblue workstation playbook

Automates user-level setup on Fedora Silverblue: **per-user** Flatpaks (under `~/.local/share/flatpak`, no sudo), CLI tools under `~/.local/bin`, and shell/editor configuration. Run Ansible **on the host**, not inside Toolbox—Flatpak tasks need the host’s `flatpak` and daemon.

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

Use the venv’s `ansible-playbook` (path below) so you do not rely on a system-wide Ansible install.

Optional: `source .venv/ansible/bin/activate` and then call `ansible-playbook` without the full path.

---

## What each role does

Use `inventory/hosts.yml` and `playbook.yml` from this repo. Pass `**--tags tag1,tag2**` (comma-separated, no spaces) to run only those roles. The included roles do **not** require sudo.

**Run all roles:**

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml
```

### bootstrap

Creates common directories: `~/.local/bin`, `~/.local/share/applications`, and `~/Applications`.

**Tag:** `bootstrap`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml --tags bootstrap
```

### starship

Downloads the Starship binary into `~/.local/bin` and adds its init block to `~/.bashrc`.

**Tag:** `starship`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml --tags starships
```

### flatpaks

Ensures the **Flathub** remote exists for **your user** (`flatpak remote-add --user`), then installs or removes Flatpaks with `**--user`** under `~/.local/share/flatpak` (no sudo).

Variables in `roles/flatpaks/defaults/main.yml`:

- `**flatpak_packages**` — list of Flatpak **app IDs** to install or uninstall (strings, e.g. `com.vscodium.codium`).

Install and remove are **separate tags** inside the role (the flatpaks role is not tagged at play level, so `--tags flatpaks_remove` does not run install tasks):

**Tag:** `flatpaks` — install everything listed in `**flatpak_packages`**

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml --tags flatpaks
```

**Tag:** `flatpaks_remove` — remove everything listed in `**flatpak_packages`**

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml --tags flatpaks_remove
```

### gnome_shell_extensions

Ensures `~/.local/share/gnome-shell/extensions` exists for per-user extensions. Installing or enabling specific extensions is left to Extension Manager (from **flatpaks**) or future automation.

**Tag:** `gnome_shell_extensions`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml --tags gnome_shell_extensions
```

### vscodium_config

Configures VSCodium `settings.json` (integrated terminal via `flatpak-spawn` into your toolbox), installs **Open VSX** extensions listed in **`vscodium_extensions`** (`roles/vscodium_config/defaults/main.yml`) via `flatpak run com.vscodium.codium --install-extension` (needs network), and adds `code` / `codium` shell aliases with **`blockinfile`**. It does **not** install the VSCodium Flatpak; the **flatpaks** role does.

**Tag:** `vscodium_config`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml --tags vscodium_config
```

### cursor

Downloads the Cursor AppImage into `~/Applications`, makes it executable, and adds a `.desktop` file under `~/.local/share/applications`.

**Tag:** `cursor`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml --tags cursor
```

### compose

Downloads the standalone **docker-compose** binary to `~/.local/bin` for use as a Podman Compose backend.

**Tag:** `compose`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml --tags compose
```

### git

Sets global `user.name` and `user.email`, and turns on **SSH** commit signing (`gpg.format` ssh and `user.signingkey`).

**Tag:** `git`

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml --tags git
```

**Set identity** (with full playbook or with `--tags git`):

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml playbook.yml \
  -e "git_user_name=Your Name" \
  -e "git_user_email=you@example.com"
```

---

Shared paths and version pins live in `vars/main.yml` and in each role’s `defaults/main.yml` where applicable.