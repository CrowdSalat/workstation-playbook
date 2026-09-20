---
name: changelog
description: Use when generating or updating CHANGELOG.md with git-cliff, preparing a release or version tag, or deciding whether a change needs a changelog entry. Trigger keywords: changelog, CHANGELOG.md, git-cliff, release notes, version bump, tag v.
---

# Changelog

Generated from git history at release time via git-cliff — never hand-edited.

## When
- At release prep: generate + commit the changelog.
- Never during dev — no `CHANGELOG.md` edits in feature commits.

## How (git-cliff)
1. Commits since the last release follow [Conventional Commits](https://www.conventionalcommits.org/).
2. `git-cliff --bump -o CHANGELOG.md` — review feat / fix / chore / break sections.
3. Commit with the bump: `docs(changelog): update changelog for <version>`.
4. `git tag v<version>`.

## Skip
- WIP / internal changes during dev — entry appears automatically at next release.
- No `CHANGELOG.md` / `cliff.toml` in the repo — propose adding one, don't improvise.