# Role: rpm_ostree

System-level role for layering packages into the Silverblue base image.

This role is intended for `system.yml` only.

## What it does

- Enables required COPR repositories
- Applies package layering using `rpm-ostree`

Requires sudo (`-K`) and a reboot after changes.

## Key variables

- `rpm_ostree_packages`: list of package maps (`name`, optional `copr`)

## Tag

- `rpm_ostree`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml system.yml -K --tags rpm_ostree
```

Then reboot:

```bash
systemctl reboot
```
