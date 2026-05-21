# Role: tool_configs

Creates configuration files for common developer tools.

## Managed files

- `~/.testcontainers.properties` (rootless Podman socket and Ryuk disablement)
- `~/.redhat/io.quarkus.analytics.localconfig` (opt out of Quarkus analytics)

## Tag

- `tool_configs`

## Run

```bash
.venv/ansible/bin/ansible-playbook -i inventory/hosts.yml user.yml --tags tool_configs
```
