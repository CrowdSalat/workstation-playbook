# Plan: Migrate Node/npm and Java tooling from nvm/SDKMAN to mise

## Objective

Consolidate per-language version managers onto a single one (`mise`), replacing:

- **`package_manager_npm`** (nvm) → mise `node` core backend + `npm:` backend
- **`package_manager_sdkman`** (SDKMAN) → mise `java` / `maven` / `kotlin` backends

Keep the existing role architecture (`tool_*` wrappers delegating to
`package_manager_*` helpers). Homebrew is explicitly **out of scope** for now,
per decision, as is anything unrelated to Java/Node cleanup.

## Background / Current State

| Concern | Current backend | mise backend |
|---|---|---|
| Node runtime (5 consumers, all `lts/*`) | `package_manager_npm` (nvm v0.40.3) | `node@lts` (core) |
| npm global packages | `npm install -g` under nvm | `npm:<pkg>@<ver>` or node `postinstall` |
| Java (Temurin 25 / 21) | `package_manager_sdkman` | `java@temurin-<ver>` (core) |
| Maven | SDKMAN candidate | `maven` (aqua:apache/maven) |
| Kotlin | SDKMAN candidate | `kotlin` (github:JetBrains/kotlin) |
| Go / Make | **already** `package_manager_mise` | `go@latest`, `make@latest` |

### Consumers

- `package_manager_npm` (via `include_role`):
  - `tool_node` (`lts/*`, empty package list) — not currently wired into either playbook
  - `tool_claude_code` (`@anthropic-ai/claude-code`)
  - `tool_opencode` (`opencode-ai`)
  - `tool_pi` (`@earendil-works/pi-coding-agent`)
  - `tool_uvcc` (`uvcc`)
- `package_manager_sdkman` (via `include_role`):
  - `tool_jvm` (java 25/21 tem, maven, kotlin)

### Execution contexts that matter

- **Fedora**: `roles_host` runs on host (all npm consumers + none of JVM);
  `roles_toolbox` (`delegate_to: toolbox_main`) runs `tool_jvm`, `tool_go`.
- **macOS**: everything runs on the host in one `roles_host` list, including
  `tool_jvm` and `tool_go`.

The `# {mark} ANSIBLE MANAGED - mise` activation block is already written into
every `shell_rc_files` entry by `package_manager_mise`. That means once Node and
Java are added to mise, their runtime availability in interactive shells comes
for free — no separate nvm/SDKMAN init blocks needed.

---

## Phase 0 — Preconditions / decisions to confirm

1. **Version selectors for Node.** Current consumers all use nvm `"lts/*"`.
   mise canonical selector is `node@lts` (supports `lts`, `lts-jod`,
   `lts/hydrogen`, ...). Confirm we keep relying on the moving LTS alias
   (`use -g node@lts`, which writes a *fuzzy* `node = "lts"`) vs. pinning a
   concrete version with `--pin`. Recommendation: keep `node@lts` fuzzy to
   match current behavior.
2. **Java version identifiers.** SDKMAN pins `25.0.3-tem` / `21.0.11-tem`.
   mise Java requires vendor-prefixed `java@temurin-25.0.3` /
   `java@temurin-21.0.11`. Confirm closest available mise artifacts (registry at
   `mise-versions.jdx.dev/tools/java`). Recommendation: keep two JDKs (a current
   LTS + a newer) mirroring today; pick the two most recent matching
   `temurin-2x` versions.
3. **Maven / Kotlin strategy.** Currently bare candidates (SDKMAN default).
   With mise, decide: `maven@latest`, `kotlin@latest`, or pin. Recommendation:
   `maven@latest`, `kotlin@latest` to mirror "default version" today.

---

## Phase 1 — Extend `package_manager_mise` for npm packages

`package_manager_mise` today only runs `mise use --global <tool>` per item
(`package_manager_mise_tools`). npm globals need the `npm:` backend.

### Changes

**`roles/package_manager_mise/defaults/main.yml`** — add:
```yaml
# npm packages to install globally, e.g. ["npm:typescript"] or
# ["npm:opencode-ai@latest"]. Semantics follow the `npm:` backend.
package_manager_mise_npm_packages: []
```

**`roles/package_manager_mise/tasks/main.yml`** — after the existing tool loop,
add a loop that installs `npm:` packages idempotently:
```yaml
- name: Install mise npm packages
  ansible.builtin.command: "{{ package_manager_mise_bin }} use --global {{ item }}"
  loop: "{{ package_manager_mise_npm_packages }}"
```

(Where `item` is an `npm:...` entry.) Reuse the same idempotency style as the
existing `mise use --global` task (rc-based CHANGED). Note: keep
`package_manager_mise_tools` for the runtime/node/java/golang entries and the
new list for `npm:` packages, so tool lists stay readable.

Optional: a dedicated `package_manager_mise` postinstall note documenting that
`npm:` packages are installed via mise's embedded `aube` (no Node needed to
install; Node needed at runtime).

---

## Phase 2 — Migrate the Node consumers

Rewrite each consumer from `package_manager_npm` to `package_manager_mise`.

### 2a. Rewrite `tool_node` (runtime only, no globals)

`roles/tool_node/defaults/main.yml`:
```yaml
---
tool_node_version: "lts"
tool_node_mise_tools:
  - "node@{{ tool_node_version }}"
```

`roles/tool_node/tasks/main.yml`:
```yaml
---
- name: Install Node.js runtime via mise package manager role
  ansible.builtin.include_role:
    name: package_manager_mise
  vars:
    package_manager_mise_tools: "{{ tool_node_mise_tools }}"
    package_manager_mise_npm_packages: []
```

### 2b. Rewrite the four tools with npm global packages

Pattern for each of `tool_claude_code`, `tool_opencode`, `tool_pi`, `tool_uvcc`:

`roles/tool_<name>/defaults/main.yml` (keep any existing node-version var):
```yaml
---
tool_<name>_node_version: "lts"
tool_<name>_node_tools:
  - "node@{{ <name>_node_version }}"
tool_<name>_npm_packages:
  - "npm:<pkg>@latest"   # pkg differs per role
```

`roles/tool_<name>/tasks/main.yml`:
```yaml
---
- name: Install Node + global npm packages via mise package manager role
  ansible.builtin.include_role:
    name: package_manager_mise
  vars:
    package_manager_mise_tools: "{{ <name>_node_tools }}"
    package_manager_mise_npm_packages: "{{ <name>_npm_packages }}"
```

Per-role `npm:` package (mirror of today's `npm install -g`):
- `tool_claude_code`: `npm:@anthropic-ai/claude-code@latest`
- `tool_opencode`: `npm:opencode-ai@latest`
- `tool_pi`: `npm:@earendil-works/pi-coding-agent@latest`
- `tool_uvcc`: `npm:uvcc@latest`

### 2c. Deprecate `package_manager_npm`

- Keep the role files but leave them unused, OR delete them. Decide below in the
  "cleanup" phase (recommendation: delete in a follow-up commit, since the role
  is not referenced in playbooks and `tool_node` no longer needs it).

---

## Phase 3 — Migrate the JVM tooling

Rewrite `tool_jvm` from `package_manager_sdkman` to `package_manager_mise`.

### Changes

`roles/tool_jvm/defaults/main.yml`:
```yaml
---
# Keys: java via mise core backend (vendor-prefixed), maven/kotlin via registry.
tool_jvm_mise_tools:
  - "java@temurin-25.0.3"
  - "java@temurin-21.0.11"
  - "maven@latest"
  - "kotlin@latest"
```

`roles/tool_jvm/tasks/install-linux.yml`:
```yaml
---
- name: Install JVM tools via mise package manager role
  ansible.builtin.include_role:
    name: package_manager_mise
  vars:
    package_manager_mise_tools: "{{ tool_jvm_mise_tools }}"
```

`roles/tool_jvm/tasks/install-mac.yml`: same `include_role` on host.

`roles/tool_jvm/tasks/main.yml`: unchanged dispatcher.

### Notes / risks

- **Version drift**: exact `temurin-25.0.3` / `temurin-21.0.11` may no longer
  be the available set. Validate against `mise-versions.jdx.dev/tools/java`
  during execution and pick the two most recent matching Temurin LTS pairs.
- **Default Java**: with SDKMAN, `tool_jvm_sdkman_defaults` set `java@25.0.3-tem`
  as default. In mise, the last `node@...`/`java@...` `mise use --global` entry
  becomes the default for that tool — ensure `java@temurin-25.0.3` is listed
  (order in the list controls default) or use an explicit
  `tool_alias`/`--pin` if a stable default name is needed.
- **Gradle caveat**: mise-installed Java toolchains are not auto-detected by
  Gradle (known upstream issue). Only relevant if the workstation runs Gradle;
  flag if that is the case.

---

## Phase 4 — Deprecate `package_manager_sdkman`

- `tool_jvm` no longer references it after Phase 3; no other consumer exists.
- Decision: delete the role (recommended) or keep as an unused orphan (not
  recommended — it would inject a stale `# ANSIBLE MANAGED - sdkman` block and a
  second `~/.sdkman` setup on fresh machines).

---

## Phase 5 — Cleanup on systems where the playbook has already run

These are **manual/host-side** steps (not part of the roles) to remove old
artifacts on an existing machine:

1. **Remove nvm init from shell configs**: the `# {mark} ANSIBLE MANAGED - nvm`
   block in `~/.bashrc` (and `~/.zshrc` on macOS). No role currently removes
   blockinfile markers for a deprecated manager.
2. **Remove SDKMAN init from shell configs**: the `# {mark} ANSIBLE MANAGED -
   sdkman` block in `~/.bashrc` / `~/.zshrc`.
3. **Remove `~/.nvm`** (nvm + all nvm-managed Node versions) once `node` is
   verified working under mise. `rm -rf ~/.nvm`.
4. **Remove `~/.sdkman`** (all SDKMAN candidates) once Java/Maven/Kotlin are
   verified under mise. `rm -rf ~/.sdkman`.
5. **Verify** after removing:

   `node -v && npm -v && java -version && mvn -v && kotlin -version`

6. **Optional housekeeping**:
   - `mise prune` periodically to drop uninstalled/old tool versions.
   - Consider `mise doctor` to confirm activation and config sanity.
   - `.nvmrc` / `.sdkmanrc` inside user projects keep working: mise natively
     reads `.sdkmanrc` and, once idiomatic file tools are enabled, `.nvmrc`
     (`mise settings add idiomatic_version_file_enable_tools node`).

> Because **none** of the current roles implement uninstall teardown, adding a
> cleanup capability to `package_manager_mise` (or a one-off `config_*`/manual
> task) is optional. Recommendation for this pass: do the shell-init + directory
> removal manually and keep roles idempotent-but-additive, matching the repo's
> existing "no teardown" convention. If a repeatable cleanup is desired later,
> file a follow-up.

---

## Phase 6 — Documentation updates

- `ROLE_CONVENTIONS.md`: replace the SDKMAN example (line 64) with a mise line,
  or document both. Update the "SDKMAN item format" section to note the mise
  equivalent, OR rename/deprecate that section.
- `package_manager_mise/README.md`: document the new `npm:` package list
  variable.
- Remove any README / role docs referencing nvm as the Node manager.

---

## Phase 7 — Validation workflow

Per `AGENTS.md` (focused runs before broad runs), use role-list overrides:

```bash
# validate a single Node consumer on the host
ansible-playbook -i inventory.yml user-fedora.yml \
  -e '{"roles_host":["tool_opencode"],"roles_toolbox":[]}'

# validate JVM inside the Fedora toolbox
ansible-playbook -i inventory.yml user-fedora.yml \
  -e '{"roles_host":[],"roles_toolbox":["tool_jvm"]}'

# macOS host
ansible-playbook -i inventory.yml user-macos.yml \
  -e '{"roles_host":["tool_jvm","tool_opencode"]}'
```

Then broaden to the full `roles_host` + `roles_toolbox` runs. Run `tool_node`
explicitly at least once (it is currently unwired in both playbooks) to confirm
the plain-runtime path.

---

## Files touched (summary)

| File | Change |
|---|---|
| `roles/package_manager_mise/defaults/main.yml` | add `package_manager_mise_npm_packages` |
| `roles/package_manager_mise/tasks/main.yml` | add npm-package install loop |
| `roles/tool_node/defaults/main.yml` + `tasks/main.yml` | nvm → mise (`node@lts`) |
| `roles/tool_claude_code/{defaults,tasks}/main.yml` | nvm → mise + `npm:` package |
| `roles/tool_opencode/{defaults,tasks}/main.yml` | nvm → mise + `npm:` package |
| `roles/tool_pi/{defaults,tasks}/main.yml` | nvm → mise + `npm:` package |
| `roles/tool_uvcc/{defaults,tasks}/main.yml` | nvm → mise + `npm:` package |
| `roles/tool_jvm/defaults/main.yml` + `tasks/install-{linux,mac}.yml` | sdkman → mise (java/maven/kotlin) |
| `roles/package_manager_npm/` | delete (deprecated) |
| `roles/package_manager_sdkman/` | delete (deprecated) |
| `ROLE_CONVENTIONS.md` | update PM examples / SDKMAN section |
| `roles/package_manager_mise/README.md` | document npm list |

## Out of scope (deferred)

- Homebrew migration/cleanup (explicitly deferred by decision).
- Full-uninstall/teardown automation inside roles (manual cleanup in Phase 5).
- Pinning concrete versions with `--pin` (kept fuzzy to match current behavior)
  unless a decision is made otherwise.
- Adding `tool_node` to the playbooks' role lists (separate decision; it is
  currently unwired).
