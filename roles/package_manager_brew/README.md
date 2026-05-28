# package_manager_brew

Uses Homebrew from the global default prefix (`/opt/homebrew` on Apple Silicon,
`/usr/local` on Intel).

## Why global Homebrew

- More source builds (fewer bottle-compatible installs) when using non-default
  per-user prefixes.
- Slower installs/upgrades and more local compile failures in non-default
  per-user prefixes.
