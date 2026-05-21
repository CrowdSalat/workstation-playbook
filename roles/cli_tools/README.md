# Role: cli_tools

Installs CLI tools into `~/.local/bin` from release binaries or archives.

## Supported entry types

Each item in `cli_tools` can be:

- direct binary download (`name`, `url`)
- archive download (`name`, `url`, `archive_binary`)

Example:

```yaml
cli_tools:
  - name: docker-compose
    url: "https://github.com/docker/compose/releases/download/v5.1.0/docker-compose-linux-x86_64"
  - name: infisical
    url: "https://github.com/Infisical/cli/releases/download/v0.43.72/cli_0.43.72_linux_amd64.tar.gz"
    archive_binary: infisical
```

## Tag

- `cli_tools`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags cli_tools
```
