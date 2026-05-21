# Role: ssh_agent

Keeps SSH agent behavior friendly across GNOME sessions, SSH sessions, TTYs, and toolboxes.

## What it does

- Keeps existing desktop agent behavior when `SSH_AUTH_SOCK` is already valid.
- Installs `keychain` script under `~/.local/bin/keychain` for fallback cases.
- Optionally manages a small `Host *` block in `~/.ssh/config` (`AddKeysToAgent`, `IdentitiesOnly`).

## Key variables

See `defaults/main.yml` for defaults; common overrides are:

- `ssh_agent_enabled` (disable whole role when `false`)
- `ssh_agent_manage_ssh_config` (skip `~/.ssh/config` block when `false`)
- `ssh_agent_identities` (defaults to `id_ed25519` keys under `~/.ssh/`)

Set overrides in `vars/local.yml`.

## Tag

- `ssh_agent`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags ssh_agent
```
