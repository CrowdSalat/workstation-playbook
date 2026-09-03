# package_manager_mise

## Role behavior

- Downloads the mise binary to `~/.local/bin` (via `curl https://mise.run | sh`).
- Adds `eval "$(~/.local/bin/mise activate <shell>)"` to each entry in
  `shell_rc_files`, using that entry's `name` for the shell.
- Runs `mise use --global <tool>` for each item in `package_manager_mise_tools`.

## Variables

| Variable                       | Default | Description                                            |
| ------------------------------ | ------- | ------------------------------------------------------ |
| `package_manager_mise_tools`   | `[]`    | Tools to pin globally, e.g. `["go@latest", "make@latest"]` |


## mise in interactive vs. non-interactive sessions

mise has two distinct ways of making tools available on `PATH`, and each fits a
different kind of session:

- **`mise activate`** — this role's default. It evaluates on every prompt
  (`eval "$(~/.local/bin/mise activate bash)"`), and only works in *interactive*
  rc files (`.bashrc`, `.zshrc`, `config.fish`, ...). Because it updates `PATH`
  reactively when the prompt is drawn, it is **not** suitable for
  non-interactive shells (a script or an IDE that never shows a prompt), nor for
  `.profile`, `.bash_profile`, or `.zprofile`.
- **Shims / `mise exec`** — for non-interactive contexts. *Shims* are symlinks
  that resolve the correct tool by inspecting `PWD` on every call. `mise exec
  -- <cmd>` runs a command with the right environment ad hoc. These are the
  options to reach into when a script, service, or editor needs a tool but does
  not source an interactive rc file.

This role uses `mise activate` because it is the recommended approach for
interactive shells. It writes the evaluation line for every entry in
`shell_rc_files` (e.g. `name: bash` -> `mise activate bash`, `name: zsh` ->
`mise activate zsh`).

### Making tools available outside interactive shells

If you need a mise-managed tool from a non-interactive script or an IDE, do
**not** add it to `mise activate`'s rc files. Instead, on the consumer side use
one of:

```sh
# shim (resolves by PWD)
mise use --global <tool>        # also creates shims under ~/.local/share/mise/shims
export PATH="$HOME/.local/share/mise/shims:$PATH"

# ad hoc
mise exec -- <command>

# current env for global tools only (no per-project switching)
eval "$(mise env)"
```

### Caveats when replacing activation with shims

`mise activate --shims` (and shims in general) trade away some features compared
to full `PATH` activation:

- Env vars defined in mise config are only exposed to mise tools, not the whole
  shell.
- Most hooks won't trigger.
- `which` points at the shim instead of the real executable.

`PATH` activation (`mise activate`) remains the recommended setup for interactive
sessions; reserve shims for the non-interactive niche.

