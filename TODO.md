# Playbook TODO

## ~~`vscodium_config` / `starship` — `lineinfile` + `block` / `marker`~~ (done)

Replaced with **`ansible.builtin.blockinfile`** so bashrc snippets are idempotent between marker lines.

Remaining ideas:

- [ ] Re-run `vscodium_config` and `starship` on a host to confirm bashrc layout and no duplicate manual aliases.

## Starship role appears to do nothing

- [ ] Investigate: **`unarchive`** used `when: starship_download.changed | default(true)` so if `get_url` reports *unchanged*, unarchive is skipped even when `~/.local/bin/starship` is missing. **`creates:`** also skips when the binary already exists (expected). Confirm tarball layout vs `--strip-components=1` on first install, and re-run with `-v` / `--diff` to see which tasks skip.
