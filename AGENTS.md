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

## Git commits

When writing commit messages (including suggested messages in chat):

- **[Conventional Commits](https://www.conventionalcommits.org/)** — use `type(optional scope): summary`. Pick a fitting `type` (`feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`, etc.); add a short `scope` when it clarifies the area (role name, component).
- **Subject** — imperative after the colon; the header line is at most **72 characters** total (including `type(scope): `), no trailing period.
- **Body** — Optional. Separate from the subject with a single blank line. Include a body only when it provides essential context for future reference (e.g., why a change was made, underlying bug root causes, or migration hints). Adopt the Linux kernel approach: use the body to explain the reasoning and intent behind the change, not to restate the code diff. Keep it tight, impactful, and free of conversational noise.
- **Footers** — none, except a **`BREAKING CHANGE:`** footer when documenting a breaking change, as Conventional Commits allows. Mark breaking commits with `!` on the type or scope when that is enough (`feat!:`, `feat(scope)!:`).
- **One logical change per commit** — do not mix unrelated topics; avoid splitting one logical change into many tiny commits.

Examples: `feat(teddycloud): add OCP entrypoint and Dockerfile`, `fix(metallb): correct BGP peer configuration`.

## Detailed conventions

See `ROLE_CONVENTIONS.md` for detailed implementation rules (naming patterns,
task file layout, toolbox model, SDKMAN item format).
