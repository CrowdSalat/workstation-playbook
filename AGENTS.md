# Agent Guidance for `playbook-next`

This file defines high-level guidance for agents working inside
`playbook-next`. It complements (does not replace) workspace-level guidance in
the repository root `AGENTS.md`.

## Scope and intent

- `playbook-next` is an architecture migration area.
- Prefer incremental, low-risk changes with real Ansible validation.
- Keep behavior readable and explicit over heavy abstraction.

## Role architecture

- Use three role families:
  - `tool_*`: user-facing tool intent (install + tool-specific config)
  - `package_manager_*`: installation mechanics and manager readiness
  - `config_*`: cross-cutting configuration not owned by one tool
- Keep `package_manager_*` roles toolbox-agnostic.
- Control toolbox execution context in playbooks (delegation), not inside PM roles.

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
