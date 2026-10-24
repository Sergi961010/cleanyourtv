---
name: Upstream Update
description: Bring Gathin23/cleanyourtv's new commits into this fork: fast-forward main, replay the personal branch, resolve the config.xml conflict, and reinstall the launcher on the TV. Use when asked to update from upstream, sync the fork, pull upstream changes, "actualiza desde upstream", "sincroniza el fork", or "trae lo nuevo del proyecto".
---

# Upstream update

`main` is a mirror of `upstream/main` and carries nothing of ours. The TV's changes
live in `personal` as exactly two commits: `config.xml` and
`examples/tcl-55v6b/debloat.txt`. Moving the mirror and replaying those two commits
is the whole job; the launcher reinstall is the last step, only when `launcher/` moved.

## Steps

1. **Preflight.** `git status --porcelain` prints nothing (ignored files aside). Then
   `git rev-parse --is-shallow-repository` must print `false`: a `true` is a leftover
   `--depth 1` clone whose history does not connect to upstream, and every rebase on it
   is nonsense. Fix with `git fetch --unshallow upstream` before going on.

2. **Fetch.** `git fetch upstream --prune --tags`.

3. **Mirror main.** `git switch main`, then `git merge --ff-only upstream/main`, then
   `git push origin main`. Done when `git rev-list --count upstream/main..main` prints 0.

4. **Replay personal.** `git switch personal`, then `git rebase main`. A conflict in
   `launcher/app/src/main/res/values/config.xml` is the normal outcome: upstream usually
   adds a key beside one of ours. **Resolution rule:** take upstream's block as the base
   and keep our value, so the new key survives and the TV's setting stays. Mark it
   resolved with `git add`, then continue with `$env:GIT_EDITOR = 'true'` set, or
   `git rebase --continue` sits waiting on an editor that never returns on Windows.
   `rerere` is on, so the same conflict resolves itself on later updates.
   Done when `git status --porcelain` is empty and `git diff --stat main..personal` shows
   exactly the two personal files.

5. **Publish.** `git push --force-with-lease origin personal`.

6. **TV, only when the launcher moved.** `git diff --stat <old main>..<new main> -- launcher/`
   tells you. If it lists anything, rebuild and install, with the toolchain this machine
   uses and the TV's address from `work/`:

   ```powershell
   $env:JAVA_HOME = 'D:\homelab-projects\android-toolchain\jdk\jdk-17.0.20.1+1'
   $env:ANDROID_HOME = 'D:\homelab-projects\android-toolchain\sdk'
   launcher\build-and-install.ps1 -TvHost <tv ip>
   ```

   Done when the script ends with `Home now resolves to com.gathin.tvhome/...` and
   `dumpsys package com.gathin.tvhome | grep lastUpdateTime` shows this minute. The
   install script leaves Home and the stock launcher exactly as the Home guard requires;
   it needs no flags here because our launcher already owns Home.

## Reference

- **A rebase that has gone wrong**: `git rebase --abort` returns to the pre-rebase
  `personal`. Nothing was pushed at that point, so nothing is lost.
- **A rebase that stops halfway**: work the conflict, `git add`, `$env:GIT_EDITOR='true'`,
  `git rebase --continue`. If the editor still blocks, `git commit --no-edit` on the
  detached HEAD finishes the current step, then `git rebase --continue` again.
- **Expected red test**: `tests/test_examples.py::test_examples_table_matches_folders`
  fails because `examples/tcl-55v6b/` has no README row. That is deliberate: leaving
  `README.md` and `examples/README.md` untouched is what keeps this rebase small, since
  upstream edits both on nearly every device list. Do not "fix" it by adding rows.
- **Branch discipline**: commits go on `personal`; `main` takes upstream commits only,
  which is what keeps `--ff-only` and GitHub's "Sync fork" working.
- **Public fork**: the TV's address, `backup/`, `shots/`, `traces/` and `work/` stay out
  of commits. `.gitignore` covers all of them; `work/HANDOFF.md` is the session note.
