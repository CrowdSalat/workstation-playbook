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
- Keep `package_manager_*` roles toolbox-agnostic.
- Control toolbox execution context in playbooks (delegation), not inside PM roles.

## Configuration file management

- When managing configuration files as complete units (no variables), store them as plain files in the role's `files/` directory (`roles/<role_name>/files/config_name.ext`) and deploy with `ansible.builtin.copy`.
- Only move files to the playbook-level `files/` directory when multiple roles need the same file.
- This enables proper syntax validation, clean git diffs, native editor support, and keeps roles modular and self-contained.
- Reserve inline content for small snippets (< 10 lines) or when the content is genuinely task-specific logic.

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

## Detailed conventions

See `ROLE_CONVENTIONS.md` for detailed implementation rules (naming patterns,
task file layout, toolbox model, SDKMAN item format).
