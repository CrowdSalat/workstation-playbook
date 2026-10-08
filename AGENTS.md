# Agent Guidance 

This file defines high-level guidance for agents working inside this workspace.

## Scope and intent

- Prefer incremental, low-risk changes with real Ansible validation.
- Keep behavior readable and explicit over heavy abstraction.

## Role architecture

- Use three role families:
  - `tool_*`: user-facing tool intent (install + tool-specific config)
  - `package_manager_*`: installation mechanics and manager readiness
  - `config_*`: cross-cutting configuration not owned by one tool
- Pick the installation channel before writing tasks. Prefer
  `tool_developer` (mise) when the tool has a mise registry entry; keep
  `package_manager_*` when a platform channel already owns the install
  (flatpak, brew, rpm-ostree, SDKMAN, pipx); otherwise write a full
  `tool_<name>` role.
- `mise` lives in `tool_developer`, not in a `package_manager_*` role — it owns
  the global `[tools]` table that its installs write to.
- mise owns the binary; config, completions, and extensions stay with the role
  that owns the tool.
- Do not create a role whose only job is adding a mise entry.
- Existing roles keep their current channel; folding one into mise is its own
  commit.
- Keep `package_manager_*` roles toolbox-agnostic.
- Control toolbox execution context in playbooks (delegation), not inside PM roles.

## Configuration file management

- When managing configuration files as complete units (no variables), store them as plain files in the role's `files/` directory (`roles/<role_name>/files/config_name.ext`) and deploy with `ansible.builtin.copy`.
- Only move files to the playbook-level `files/` directory when multiple roles need the same file.
- This enables proper syntax validation, clean git diffs, native editor support, and keeps roles modular and self-contained.
- Reserve inline content for small snippets (< 10 lines) or when the content is genuinely task-specific logic.

## Agent harness content

- Central `AGENTS.md` (`roles/config_agents_harnesses/files/AGENTS.md`) carries only always-on guardrails: git commit discipline, code quality/safety, editing behavior. Everything procedural belongs in skills.
- Skills live in `roles/config_agents_harnesses/files/skills/<name>/SKILL.md` and hold on-demand how-to playbooks (containers, changelog, openshift-deploy, tool-installation).
- Skill style is brief and checklist-like:
  - Imperative bullets and exact commands over prose.
  - Keep every hard constraint, file path, and copy-paste template (e.g. workflow YAML).
  - No lead-in sentences; state load-bearing "why" only where it changes behavior.
  - Frontmatter: `name` matches the folder; `description` in third person, what + when, front-loaded trigger keywords.
- Deploy via focused run of `config_agents_harnesses`; verify content landed in `~/.agents/skills` and `~/.claude/skills` and the central `AGENTS.md` was updated.

## Playbook execution model

- Fedora uses split role lists:
  - `roles_host`
  - `roles_toolbox`
- macOS uses host list:
  - `roles_host`
- Toolbox prerequisites in playbook tasks should run only when
  `roles_toolbox | length > 0`.

## Editing and safety

- Do not revert unrelated user changes.
- Avoid destructive git operations.
- Keep idempotency in mind for all tasks.
- Prefer explicit defaults and clear variable naming.

## Validation workflow

- Run syntax checks after structural changes.
- Prefer focused test runs using role-list overrides to isolate behavior.
- Escalate to broader runs only after focused runs pass.
- After editing files that `config_agents_harnesses` deploys (the central
  `AGENTS.md`, skills), tell the user to redeploy the role; verify the change by
  reading the repo source, not the deployed copy.
- Deleting a role that once wrote a `blockinfile` marker leaves that marker
  behind in shell rc files. Flag it for the user to clean up by hand rather than
  reaching into their home directory.

## Detailed conventions

See `ROLE_CONVENTIONS.md` for detailed implementation rules (installation
channel selection, naming patterns, task file layout, toolbox model, SDKMAN
item format).
