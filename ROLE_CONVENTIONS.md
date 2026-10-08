# Role conventions for `playbook`

Use these conventions for `tool_<name>` roles.

## Choosing an installation channel

Decide the channel before writing any task. It is the first question, not an
implementation detail to settle once tasks exist.

1. **Does the tool have a mise registry entry?**

   ```bash
   mise registry | grep -E '^<tool>[[:space:]]'
   ```

   A hit means mise can own the binary: add `<tool> = "latest"` to
   `roles/tool_developer/files/config.toml` and stop there.

2. **Does a platform channel already own the install?** If the role goes
   through `package_manager_flatpak`, `package_manager_brew`,
   `package_manager_rpm_ostree`, `package_manager_sdkman`, or
   `package_manager_pipx`, it stays there. mise does not replace flatpak, Homebrew
   casks, COPR layering, SDKMAN candidates, or pipx apps.

3. **Otherwise: a full `tool_<name>` role.** Use the vendor's supported path —
   a `package_manager_*` helper when one fits, or direct modules (`get_url`,
   `unarchive`, `uri`) for vendor installer scripts and release tarballs.

`tool_developer` is the only mise entry point. It owns the mise bootstrap, shell
activation, and the global `[tools]` table. There is deliberately **no**
`package_manager_mise` role; the earlier one was removed as unreferenced.

### mise owns the binary, a role owns the config

mise declares versions and installs binaries. It configures nothing. So a tool
whose configuration this playbook owns needs both halves:

- the `[tools]` entry in `roles/tool_developer/files/config.toml`
- a `tasks/postinstall-<tool>.yml` inside `tool_developer`, wired from its
  `tasks/main.yml`, for every config file, completion, extension, or harness
  detector directory

That is how `gh`, `opencode`, and `pi` are handled: mise installs the binary,
`tool_developer/tasks/postinstall-*.yml` owns the rest.

A role whose only remaining job is configuration still stays a role. `tool_git`
and `tool_vi` install nothing and are correct as they are.

Do not create a `tool_<name>` role whose only job is to add a mise entry. That
is exactly the shape `tool_go`, `tool_gh`, `tool_opencode`, and `tool_pi` had
before they were folded into `tool_developer`.

### New work is forward-only

Existing roles keep the channel they have. `tool_claude_code` stays on
`package_manager_npm`, `tool_jvm` on `package_manager_sdkman`,
`tool_pre_commit` on `package_manager_pipx`. Folding one of them into mise is a
deliberate refactor in its own commit, never a side effect of adding an
unrelated tool.

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
- CLI completion belongs in the role that owns the tool. For a mise-installed
  tool that is `tool_developer/tasks/postinstall-<tool>.yml`; for a role-owned
  install it is that role's postinstall.

## Playbook registration

- Register new roles in the playbooks:
  - `user-fedora.yml`: `roles_host` for host tools, `roles_toolbox` for toolbox
    tools.
  - `user-macos.yml`: `roles_host` only (no toolbox).
- Toolbox context also needs `toolbox_name`; toolbox prerequisites run only when
  `roles_toolbox | length > 0`.
- Keep related roles grouped in the lists (for example cloud CLIs together).

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
- `mise` is deliberately **not** one of these roles. It lives in `tool_developer`
  because it also owns the global `[tools]` table that its installs write to.
  See "Choosing an installation channel" above.

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
