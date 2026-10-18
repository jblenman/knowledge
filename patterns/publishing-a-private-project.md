# Publishing a private working repository as a public project

Procedure used for `kids-os` (2026-10-18): a repo built over two days with the owner's user name, SSH keys, LAN addresses and family notes in the tree, turned into a project a stranger can build — without maintaining a sanitized copy.

## Principle: make the project generic, don't sanitize a copy
- A second "clean" copy drifts within a week. Instead move every personal value out of the tree into a **per-user configuration directory** (`~/.config/<project>/`) and make the tree read it: scalars through a `KEY=VALUE` env file passed by *name* into the build (no value on a command line or in logs), files (ssh keys, app lists, wallpapers) **staged into a git-ignored path inside the tree** for the duration of the build and consumed by the post-install step, then removed.
- Defaults must be sensible without the directory at all (admin `admin`, UTC, no Wi-Fi), and the build must *warn* when a missing file locks the user out (no `authorized_keys` → no ssh).
- Scripts take the owner's identity from the same file (`lib/env.sh`: environment → secrets file → default), so test harnesses and deploy scripts carry no names either.

## Gates before the first public push
1. **Grep gate** over the whole tree (tracked and untracked): user names, LAN addresses (`192\.168\.`), hostnames, Wi-Fi names, family words, machine names, internal tool names. Expect only the LICENSE copyright line.
2. **History gate**: `git log -p --all | grep` for the same list plus secrets (`PSK=`, `$6$` hashes, `PRIVATE KEY`). If anything personal ever landed in history, the public repo gets a **fresh history** (below) rather than a rewrite.
3. **Verification gate**: build with a *non-personal* configuration (generic admin name, test PIN, UTC) and run the project's real tests on that build — publish what was proven, not what was edited.

## Publishing with a fresh history, keeping the old one
```
git add -A && git commit -m "Make the project generic"        # last commit of the private history
git push origin main
gh repo rename <name>-home -R <user>/<name> --yes               # the private repo becomes the archive
git remote rename origin archive; git remote set-url archive https://github.com/<user>/<name>-home.git
git branch -m main history                                      # old history stays checked out locally as `history`
git checkout --orphan main && git add -A && git commit -m "<name>: first public release"
gh repo create <user>/<name> --public --source=. --remote=origin --push --description "…"
git branch --set-upstream-to=archive/main history
```
- One checkout, two remotes: `origin` (public, day-to-day) and `archive` (private, frozen). Nothing destructive: the private repo is renamed, not rewritten; GitHub redirects its old URL.
- Verify from GitHub's side afterwards: `gh api repos/<user>/<name>/git/trees/main?recursive=1 --jq '.tree[].path'` (quote the URL — zsh globs the `?`), license detection, the README as served, the commit author (the owner's name and email become public; that is their choice to make).

## What the public README needs that the private one did not
Who it is for and the maturity level, requirements (target hardware, build host), a quick start that works from a fresh clone, every configuration key in one table, a "day to day" section, honest limits (what is *not* a security boundary), the architecture in one screen, a development/testing page, a hardware-notes page for the quirks met, a license line that also names third-party components downloaded at build time.
