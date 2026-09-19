# Role conventions for `playbook-next`

Use these conventions for `tool_<name>` roles.

## Priorities

- Prefer readability over runtime micro-optimizations.
- Playbooks are run infrequently, so clear role flow is more important than
  avoiding a few repeated idempotent checks.

## Baseline

- Always have `tasks/main.yml`.
- For each phase, choose one style:
  - shared file (`tasks/install.yml` or `tasks/postinstall.yml`)
  - OS files (`tasks/install-linux.yml` + `tasks/install-mac.yml`, same for postinstall)

## When to split by OS

Split into OS-specific files only when branching starts to dominate readability.

Examples:

- `tasks/install-linux.yml`
- `tasks/install-mac.yml`
- `tasks/postinstall-linux.yml`
- `tasks/postinstall-mac.yml`

It is valid to split only one phase:

- split `install`, keep `postinstall` shared
- or split `postinstall`, keep `install` shared

## Practical rule

If many tasks in `install.yml` or `postinstall.yml` are wrapped in OS `when`
conditions, split that phase into platform files and include them directly from
`main.yml`.

## No thin dispatcher phase files

- Do not keep `install.yml` or `postinstall.yml` as thin dispatchers.
- If phase logic is split by OS, include `install-linux.yml` /
  `install-mac.yml` (or postinstall equivalents) directly from `main.yml`.

## Tool role ownership

- Keep each tool role self-contained.
- A tool role should describe its own install + postinstall behavior end-to-end.
- Use package manager roles as helpers, but keep control flow in the tool role.
- CLI completion belongs in `tool_*` roles when available for that tool.
  Prefer enabling completion during tool postinstall so completion config stays
  close to the owning tool behavior.

## Agent harness configuration ownership

- Shared agent rules and skills for AI coding harnesses live in
  `config_agents_harnesses`, not in `tool_*` roles.
- `tool_*` roles only scaffold their harness config directory; the config role
  deploys the shared `AGENTS.md` content to whichever harnesses are installed
  (detected by existing config directory) plus universal skills to
  `~/.agents/skills`.
- `config_agents_harnesses` must run after the installed `tool_*` harness roles
  in `roles_host`, because target directories are the detection signal.
- Tool-specific config that a role already owns (for example
  `~/.claude/settings.json` in `tool_claude_code`) stays in that role.

## Package manager role behavior

- Package manager roles should do both:
  - fast precheck/setup for manager readiness
  - idempotent package install for requested packages
- Keep behavior explicit per manager:
  - `package_manager_flatpak`: check presence, then configure remote and install packages
  - `package_manager_brew`: bootstrap Homebrew when missing, apply shellenv setup, then install packages
  - `package_manager_rpm_ostree`: check presence, configure COPR repos, then layer packages and report reboot need
  - `package_manager_sdkman`: bootstrap SDKMAN when missing, ensure shell init blocks (`.bashrc` and `.zshrc`), then install candidates
  - `package_manager_pipx`: bootstrap pipx when missing, ensure application path, then install packages
- Tool roles may call package manager roles repeatedly; this is acceptable for
  readability.

## Toolbox execution context

- Treat toolbox as an execution-context layer on Linux, not as a normal tool.
- Package manager roles stay toolbox-agnostic.
- Playbook controls toolbox execution via delegated role calls.
- `toolbox_name` is set at playbook scope for Linux playbooks.

## SDKMAN item format

- Use a single list variable with one of:
  - `candidate` (install default version)
  - `candidate@identifier` (install exact version identifier)
- For Java, identifier includes vendor/distribution suffix (for example
  `21.0.8-tem`), so prefer explicit identifiers when pinning versions.
