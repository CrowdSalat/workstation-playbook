# tool_ssh

Owns the SSH client half of a workstation: key material, `~/.ssh/config`, and
nothing else. `ssh` and `ssh-keygen` come from the OS; this role provisions and
configures.

Must run **before** `tool_git` in `roles_host`, because `tool_git` reads the
signing key this role publishes as the `workstation_ssh_git_public_key` fact.

## Key naming

Keys are `id_<type>_<purpose>` — `id_ed25519_github`, `id_ed25519_openshift`,
`id_ed25519_oracle-vls`. The purpose suffix is not cosmetic: a default-named
`id_ed25519` is offered implicitly by every host, while a purpose-suffixed key
is only ever offered where `~/.ssh/config` names it.

Adding a purpose means editing both `tool_ssh_identities` and
`files/ssh_config`. Those two must agree or the key is generated but unused.

## Key comments

Derived as `<key name>@<user>@<host>_<YYYY-MM-DD>`:

```
id_ed25519_github@jan@fedora_2026-10-09
```

**The format contains no spaces, deliberately.** GitHub derives an automatic key
title from the comment when a key is uploaded and truncates at the first space,
so a space-separated date would be silently dropped from the key title while
still looking correct locally in `ssh-add -l`.

Set `tool_ssh_key_comment` to override, which applies verbatim to every
identity.

`key_result` only exists inside the generate loop, so the comment is built in
`tasks/keys.yml` rather than `defaults/main.yml`.

Comments are free-form metadata that no part of SSH resolution reads. They are
worth populating because the `.pub` file is what GitHub, OpenShift and Oracle
display as the key title, where the filename is invisible — the comment is the
only provenance that travels with the key. The date makes a key from a retired
machine identifiable later. Existing keys are never rewritten, so each key keeps
the date of its original provisioning run.

The comment is stored in the OpenSSH-format private key as well as the `.pub`,
but it sits inside the base64 armor, so a plaintext search of the private key
finds nothing. With a passphrase set it is inside the encrypted section and
unrecoverable. Read comments from the `.pub`, which is never encrypted.

Hosts that show key titles truncate long comments, so this format is
deliberately short. Provenance that would push it further — playbook name, git
commit — is available via `tool_ssh_key_comment` if you want it.

## Never overwritten

Generation is guarded on the private key existing. Re-running is a no-op, and a
key you restored from a backup or a password manager is never clobbered. There
is no rotate path by design; delete the key file by hand to regenerate.

## Passphrases

Default is empty, relying on the platform agent (GNOME on Fedora, launchd on
macOS) to cache the key for the session. This role does not run keychain, so a
non-empty `tool_ssh_key_passphrase` needs an agent that prompts, or you get
prompted on every use.

## `~/.ssh/config` ownership

The role owns the whole file and overwrites it every run. Keep custom stanzas in
`roles/tool_ssh/files/ssh_config`; do not edit `~/.ssh/config` directly.

`IdentitiesOnly yes` is set per host, never on `Host *`. When set, ssh offers
only configured or default-named key files and ignores the agent, so a host
without an explicit `IdentityFile` stanza would silently get no key.

## GitHub registration

`tool_git` enables `gpg.format=ssh` with `commit.gpgsign=true`. One public key
needs registering in two GitHub slots: *Authentication keys* for `git@` push,
and *Signing keys* for the verified badge. The role prints both when it creates
a key; it does not touch GitHub itself.

## Stale marker

Replacing the old `tool_ssh_agent` role leaves an `# BEGIN ANSIBLE MANAGED -
ssh_agent` block in shell rc files. It is removed by hand, not by this role.
