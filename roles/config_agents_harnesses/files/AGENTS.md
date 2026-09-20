# Global Agent Rules

This file defines global guidance for AI coding agents (Claude Code, Pi, Cursor, etc.).
These rules apply across all projects unless overridden by project-specific AGENTS.md files.

## Git Commits

Keep commits atomic: commit only the files you touched and list each path explicitly. For tracked files run `git commit -m "<scoped message>" -- path/to/file1 path/to/file2`. For brand-new files, use the one-liner `git restore --staged :/ && git add "path/to/file1" "path/to/file2" && git commit -m "<scoped message>" -- path/to/file1 path/to/file2`

When writing commit messages (including suggested messages in chat):

- **[Conventional Commits](https://www.conventionalcommits.org/)** — use `type(optional scope): summary`. Pick a fitting `type` (`feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `chore`, `ci`, `build`, etc.); add a short `scope` when it clarifies the area (role name, component).
- **Subject** — imperative after the colon; the header line is at most **72 characters** total (including `type(scope): `), no trailing period.
- **Body** — Optional. Separate from the subject with a single blank line. Include a body only when it provides essential context for future reference (e.g., why a change was made, underlying bug root causes, or migration hints). Adopt the Linux kernel approach: use the body to explain the reasoning and intent behind the change, not to restate the code diff. Keep it tight, impactful, and free of conversational noise. **Never hallucinate reasons or include placeholders (like `TODO` or `[Fix ID]`).**
- **Footers** — none, except a **`BREAKING CHANGE:`** footer when documenting a breaking change, as Conventional Commits allows. Mark breaking commits with `!` on the type or scope when that is enough (`feat!:`, `feat(scope)!:`). **Use standard issue closing keywords (e.g., `Closes #123` or `Fixes PROJ-456`) on a separate line at the bottom if an issue/ticket exists.** Never add `Co-Authored-By:` or other attribution footers — git history tracks authorship automatically.
- **No Agent Meta-Language** — Do not include conversational filler or references to your identity as an AI (e.g., avoid *"As requested,"* *"Per instructions,"* or *"I have fixed..."*). The commit must read as if written by a core maintainer.
- **One logical change per commit** — do not mix unrelated topics; avoid splitting one logical change into many tiny commits. **If a tool-assisted change affects many files (e.g., a global renaming or dependency bump), explicitly state the macro-reason in the body.**

Examples: 
- `feat(teddycloud): add OCP entrypoint and Dockerfile`
- `fix(metallb): correct BGP peer configuration`
- `docs(readme): document new environment variables`

## Code Quality & Safety

- **Idempotency** — Keep idempotency in mind for all tasks and scripts.
- **No destructive git operations** — Avoid `git reset --hard`, `git push --force` to main/master, or rewriting published history without explicit user confirmation.
- **Explicit over implicit** — Prefer explicit defaults and clear variable naming.
- **Security** — Never commit secrets, API keys, or credentials to version control.

## Editing Behavior

- Do not revert unrelated user changes.
- When making structural changes, validate syntax before committing.
- Prefer focused, incremental changes over large refactors.
