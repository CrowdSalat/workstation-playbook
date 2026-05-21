# Role: git

Configures global Git identity and SSH commit signing.

## What it sets

- `user.name`
- `user.email`
- `gpg.format ssh`
- `user.signingkey`

Values come from `vars/local.yml` (git-ignored). Use `vars/local.yml.example` as template.

## Tag

- `git`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags git
```
