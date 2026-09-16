# mise in non-interactive contexts (cron, systemd, agent harnesses)

Empirical findings plus mise's documented intent for contexts where no prompt is
ever displayed: cron, systemd user units, CI, and agent harnesses.

## Why `mise activate` cannot cover these

`mise activate` recomputes PATH/env **when the shell prompt is displayed**.
Non-interactive shells (`bash -c`, crons, systemd, agent exec) never display a
prompt, so activation never runs and PATH is never updated. This is stated in
mise's own troubleshooting docs; shims, `mise exec`, `mise run` and `mise env`
are the supported replacements.

## Empirical matrix (Fedora, bash)

| Shell type | Reads | `bat` found? |
|---|---|---|
| `bash -c` (child of a mise-activated session) | nothing | yes — **inherited** mise PATH |
| `bash -lc` (login) | `~/.bash_profile` → `mise activate --shims` | yes |
| `bash -i` (interactive) | `~/.bashrc` → `mise activate` | yes |
| `bash -c`, **clean env** (cron/systemd-style) | nothing | **no** — `bat` and even `mise` off PATH |

The middle two rows come from `tool_developer` writing `activate` into rc files
and `activate --shims` into profile files — matching mise's recommended split.
Row 1 only worked because the parent session had injected mise into PATH; that is
not something automation should rely on. Row 4 is the real gap.

## mise's documented options for non-interactive contexts

1. **Shims** — `export PATH="$HOME/.local/share/mise/shims:$PATH"` or `mise
   activate --shims`. mise docs' recommendation for CI, scripts and IDEs. A shim
   resolves the tool + env for the current directory *each time it runs*, so it
   survives `cd`. Env vars are only set when a shim is actually called; the
   `cd`/`enter`/`leave` hooks and `watch_files` require `activate`.
   `not_found_auto_install` (default) keeps shims as an auto-install fallback.
2. **`mise x|exec`** — load the full mise env, run one command. The explicit,
   pristine contract: no rc modification needed, nothing on PATH.
3. **`mise r|run`** — tasks. jdx's stated recommendation: "PATH for local
   development, shims for IDE stuff, tasks for scripts and CI/CD." Tasks use the
   tools declared in `mise.toml` without any shell integration.
4. **`mise env`** — dump env vars, but only for the *global* toolset; it does not
   adjust per project.
5. **`eval "$(mise activate bash)"; eval "$(mise hook-env)"`** — manual hook
   invocation; must be repeated after each `cd`.
6. **systemd user units** — mise can *declare* units in config
   (`[bootstrap.linux.systemd.units]`) and converge them with `mise bootstrap
   linux systemd-units apply`. User-only (`~/.config/systemd/user/dev.mise.*`),
   `systemctl --user` managed, `~` expansion in `exec_start`. Crucially, the unit
   needs an explicit `environment = { PATH = "..." }` because systemd never
   sources any shell rc/profile.

## Recommended contract per context

- **cron** — prefer `$HOME/.local/bin/mise exec -- <cmd>` in the crontab, or a
  small wrapper that adds the shims dir + `~/.local/bin` to PATH and then execs.
  Never assume a bare `bash -c` from cron reads profiles.
- **systemd user unit** — if declared via mise bootstrap, set
  `environment = { PATH = "$HOME/.local/bin:/usr/local/bin:/usr/bin:/bin:$HOME/.local/share/mise/shims" }`
  explicitly. `ExecStart = "$HOME/.local/bin/mise exec -- <cmd>"` also works and
  keeps the unit self-explanatory.
- **agent harness** — an agent shells out to tools (gh, terraform, bat, …) it
  expects on PATH. Two robust options:
  - *Recommended:* the harness starts with the shims dir (+ `~/.local/bin`)
    exported in its own environment, so every `bash -c` subprocess inherits it.
    Mirrors `<shims> >> $GITHUB_PATH` in CI.
  - The harness launches login shells (`bash -lc`/SSH-style), which pick up the
    profile shims automatically.
  - Keep `mise exec` as the escape hatch for one-off pinned invocations.
- **CI** — `echo "$HOME/.local/share/mise/shims" >> $GITHUB_PATH` or prefix with
  `mise exec`.

## Current playbook state and gaps

- `tool_developer` writes `mise activate` (rc) + `mise activate --shims`
  (profile). So interactive and login shells are covered.
- **Gap:** bare `bash -c` in a clean env (cron, systemd, some agent exec paths)
  finds neither tools nor mise. Everything else inherits PATH from the harness
  or a profile.
- Candidate hardening (not yet implemented): a wrapper/fragment that exports
  shims into non-login, non-interactive contexts where the harness controls the
  environment, plus documented `mise exec`-based cron/systemd examples.