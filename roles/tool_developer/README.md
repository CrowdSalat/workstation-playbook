# tool_developer

Standalone mise bootstrap role: installs mise, wires shell integration, and
manages a static global `files/config.toml`. Deliberately does **not** delegate
to a `package_manager_*` role; it is the self-contained mise setup.

## Role behavior

- Installs `mise` to `~/.local/bin` (`curl https://mise.run | sh`) if missing.
- Adds `eval "$(~/.local/bin/mise activate <shell>)"` to each interactive rc file
  (`.bashrc`, `.zshrc`) — the PATH solution.
- Adds `eval "$(~/.local/bin/mise activate <shell> --shims)"` to each profile
  file (`.bash_profile`, `.zprofile`) — non-interactive/login coverage.
- Deploys the global config (`~/.config/mise/config.toml`) from a static
  `files/config.toml` dump, then runs `mise install` and verifies a tool.

## Variables

| Variable                 | Default                                   | Description                       |
| ------------------------ | ----------------------------------------- | --------------------------------- |
| `tool_developer_home`    | `{{ ansible_facts['env']['HOME'] }}`      | Home to wire shell files under.   |

## The mise integration model (gist of the design discussion)

mise has three tiers, and it matters which context you are in:

| Context | Shell init used | How tools resolve |
| --- | --- | --- |
| Interactive terminal (`bash -i`) | `.bashrc` → `mise activate` | prompt-driven PATH update by mise |
| Login / non-interactive-login (`bash -lc`) | `.bash_profile` → `mise activate --shims` | **shims** (resolve per `PWD` on every call) |
| Script child of an activated shell (`bash -c`) | none | inherits PATH from the parent (works, but not a contract) |
| Clean non-interactive (`bash -c` w/ empty env — cron, systemd, some daemons) | none | **nothing** — neither `mise` nor shims on PATH |

Key facts learned empirically and from mise docs:

- `mise activate` only updates PATH **when the prompt is drawn**, so it is
  useless where no prompt exists. That is why `activate` goes in rc files and
  `activate --shims` in profile files — this role implements exactly that split.
- A bare `env -i bash -c 'bat --version'` fails (rc 127): cron/systemd-style
  exec does **not** read `.bash_profile`/`.bashrc`, so login shims don't apply.
- Shims are the right tool for non-interactive exec environments (CI, IDEs,
  editors); they are symlinks into the mise binary and resolve by `PWD` every
  call. mise's docs recommend them for CI (`<shims> >> $GITHUB_PATH`).

### cron

Crontab runs `/bin/sh -c "<cmd>"` with a near-empty env and no rc/profile
sourcing. Wrap commands explicitly:

```bash
$HOME/.local/bin/mise exec -- <command>
```

or export the shims dir in the wrapper/crontab PATH.

### systemd (user units)

systemd `execve`s the binary directly — no shell, no profiles. If tools come
from mise, state the PATH in the unit explicitly:

```ini
Environment=PATH=/home/<user>/.local/bin:/usr/local/bin:/usr/bin:/bin:/home/<user>/.local/share/mise/shims
ExecStart=/home/<user>/.local/bin/mise exec -- <command>
```

mise can also *declare* user units in config under `[bootstrap.linux.systemd.units]`
and converge them with `mise bootstrap linux systemd-units apply` (user-only,
`systemctl --user`, `~` expansion in `exec_start`, but still needs an explicit
`environment` because systemd sources nothing).

### Agent harnesses (opencode / claude / pi, shells via `bash -c`)

Works today because the harness is **spawned from an interactive shell** and
inherits its activated PATH. If an agent were ever daemonized (systemd, launchd,
editor service) with a clean env, its `bash -c` children would find nothing; the
fix is to export shims (+ `~/.local/bin`) in the harness's own environment, the
same pattern as CI.

## Notes / caveats

- The global config is a static dump; manual `mise use -g` writes are
  overwritten on the next run. Edit `files/config.toml` instead.
- Interactive shells keep shims on PATH as an auto-install fallback behind the
  tool paths (`not_found_auto_install` default).

## Troubleshooting

### `mise ERROR No version is set for shim: <tool>`

The tool is installed and reachable (the shim fired), but **no version is
configured for the context the shim ran in**. Shims resolve by walking up from
their `PWD`, so this error means no `mise.toml` in the current directory
hierarchy declares the tool — and no global default exists either. Reported by
agent harnesses as "tool not found"; it is a config problem, not a missing
binary.

Fixes, by intent:

- **Project-scoped:** declare it in the project's `mise.toml` (`[tools]
  golangci-lint = "2.13.2"`) and run from inside that directory — the agent
  must exec with cwd under the project, not a temp dir or the repo root.
- **Global default:** add it to `roles/tool_developer/files/config.toml` so the
  playbook owns it. A bare `mise use -g <tool>@<ver>` works but is overwritten
  on the next run.
- **Ad-hoc:** `mise exec -- <tool> ...` when the cwd can't be guaranteed.