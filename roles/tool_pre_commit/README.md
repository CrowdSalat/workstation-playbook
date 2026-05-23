# tool_pre_commit

Installs the `pre-commit` CLI for the user.

## Quick start

`pre-commit` is configured per repository:

```bash
# In your git repository:
pre-commit sample-config > .pre-commit-config.yaml
pre-commit install
pre-commit run --all-files
```

After this, checks run automatically on each commit in that repository.
