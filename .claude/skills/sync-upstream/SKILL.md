---
name: sync-upstream
description: Sync this personal Godot fork with upstream godotengine/godot. Fetches upstream, fast-forwards master, merges it into the custom branch, rebuilds the Windows editor, smoke tests it, and reports what came in. Use this whenever asked to sync, update, pull, merge or catch up with upstream Godot, to "get the latest Godot", to "update the fork", or to see how far behind the fork is, even when the word "sync" is not used.
---

# Sync the Godot fork with upstream

Brings the fork up to date with `godotengine/godot` and proves the result
still builds and runs. The value of this routine is the guardrails: a sync
that silently loses a customization, or leaves a fork that no longer builds,
is worse than no sync.

"The maintainer" below is the person you are working with.

## Where things are

Paths are relative to the repository root, which is where to run everything.

- `upstream` = godotengine/godot, `origin` = the maintainer's fork
- `master` mirrors `upstream/master`. It never carries the fork's own
  commits and is never checked out.
- `custom` carries everything that belongs to the fork: engine changes,
  `CLAUDE.md`, the `fork\` docs and this skill. It stays checked out for the
  whole sync, because switching away would remove those files from disk.
- Human-facing version of this procedure, with recovery steps:
  `fork\upstream-sync.md`

Use the PowerShell tool. Git Bash sets `MSYSTEM`, which makes Godot's build
scripts look for dependencies in the wrong place. Use `py -m SCons`, because
`scons` is not on PATH.

## Modes

Read the arguments passed, if any:

- No arguments: full sync (steps 1 to 7), without pushing.
- `check`: do steps 1 to 3, report, and change nothing beyond the fetch.
- `push`: full sync, then push the updated branches in step 6.

Pushing is opt-in because it changes the public GitHub fork. If the
maintainer asked for a push in plain words instead of the argument, that
counts too.

## 1. Preflight

```powershell
git status --porcelain            # must print nothing
git branch --show-current         # must print custom
git remote get-url upstream       # must be godotengine/godot
Test-Path .git\MERGE_HEAD         # must be False
```

If the tree is dirty or a merge is already in progress, stop and say what is
there. Do not stash or discard it: uncommitted work in an engine fork is
usually a half-finished customization, and only the maintainer knows whether
it should be kept.

If another branch is checked out, stop and ask before switching. The
maintainer may be partway through a pull request branch.

## 2. Fetch

```powershell
git rev-parse master      # note this hash as <old>
git fetch upstream
```

Shell variables do not survive between tool calls, so keep the `<old>` hash
and paste it into later commands.

## 3. Assess

```powershell
git rev-list --left-right --count master...upstream/master
git rev-list --count custom..master
```

The first command prints commits only on `master`, then commits only on
upstream. The second prints how many `master` commits `custom` still lacks.

- Left number above 0: something was committed to `master` directly. Stop.
  Fixing it means moving `master`, which can lose commits, so show the stray
  commits (`git log --oneline upstream/master..master`) and point to "If the
  fast-forward is rejected" in `fork\upstream-sync.md`. Proceed only on a
  clear go-ahead.
- Right number 0 and second command 0: nothing to do. Report "already up to
  date" and skip the build unless `bin\godot.windows.editor.x86_64.exe` is
  missing.
- Right number 0 but second command above 0: skip to the merge in step 4.
- In `check` mode, stop here and report the counts plus the highlights
  described in step 7, reading them from `master..upstream/master`.

## 4. Update the branches

```powershell
git fetch . upstream/master:master
git merge --no-edit master
```

The first command moves the local `master` branch to match upstream without
checking it out. It only fast-forwards: if it prints
`[rejected] (non-fast-forward)`, treat it like the "left number above 0" case
in step 3. The second merges into `custom`, the checked-out branch.

**When the merge conflicts**, list the files with
`git diff --name-only --diff-filter=U` and read each conflict. `rerere` is
enabled, so some may already be resolved from a previous sync; those still
need a look and a `git add`.

Resolve by keeping upstream's new structure and re-applying the fork's change
on top of it. Taking the old customized side wholesale tends to revert
upstream fixes without anyone noticing.

Resolve it yourself when the intent is obvious, for example both sides added
adjacent lines or upstream renamed something the customization uses. Ask the
maintainer when upstream rewrote or deleted the code the customization lived
in, because then the customization has to be redesigned and that is their
call. When asking, leave the merge in progress, show the conflict, and say
that `git merge --abort` puts everything back.

## 5. Build and smoke test

```powershell
py -m SCons platform=windows *> <scratchpad>\sync-build.log
```

A build after a sync takes anywhere from about 40 seconds to about 5 minutes
depending on how much upstream changed. Run it in the background and read the
log tail when it finishes. Judge success by `$LASTEXITCODE` being 0 and the
line `scons: done building targets.`, not by PowerShell's red error
formatting, which it applies to harmless stderr lines such as
`WARNING: msgfmt not found`.

Then:

```powershell
.\bin\godot.windows.editor.x86_64.console.exe --version
.\bin\godot.windows.editor.x86_64.console.exe --headless --quit
```

Both must exit 0. The version string ends in a short commit hash, and it
should match the start of `git rev-parse HEAD`. A mismatch means the binary
is stale and the build did not really run.

Capture the output into a variable and read `$LASTEXITCODE` straight after.
Piping the exe into `Select-Object -First N` ends the pipeline early, kills
the process and reports exit -1, which looks like a failed smoke test.

**When the build fails**, work out which kind of failure it is before fixing
anything:

| What the log says | What to do |
|---|---|
| D3D12, Mesa, AccessKit or ANGLE missing or wrong version | Upstream bumped a dependency. Run the matching `py misc/scripts/install_d3d12_sdk_windows.py`, `install_accesskit.py` or `install_angle.py`, then rebuild |
| Compile error in a file the fork modifies (`git diff --name-only master...custom`) | Upstream changed an API the customization uses. Fix the customization if the change is mechanical; otherwise show the maintainer |
| Compile or link error in untouched upstream code | Probably stale objects. Run `py -m SCons platform=windows --clean` and rebuild once. If it still fails, report it; upstream `master` is occasionally broken |

Do not push a branch that fails to build.

## 6. Push, only if asked

```powershell
git push origin master custom
```

Without a push request, leave both branches local and say so in the report.

## 7. Report

Report in this shape:

```
Result: synced | already up to date | stopped: <reason>
Upstream: <old short hash>..<new short hash>, <N> pull requests, <first date> to <last date>
Custom branch: merged cleanly | merged with conflicts in <files> | nothing to merge
Build: passed in <time> | failed: <one line> | skipped: <why>
Smoke test: <version string>, headless start OK
Pushed: yes | no (run `git push origin master custom` to publish)

Worth knowing:
- <a few upstream changes that matter to the maintainer>
```

For the "Worth knowing" list, read what came in:

```powershell
git log --first-parent --format='%h %cs %b' <old>..master
git diff --stat <old>..master -- SConstruct platform/windows misc/scripts
```

Every commit on upstream `master` is a merged pull request whose body is the
pull request title, so the first command gives a readable changelog. Pick
out what affects this fork rather than listing everything:

- changes to the build system or the `install_*.py` dependency versions
- changes to files the fork also modifies, since those are where the next
  conflict will come from
- breaking changes and anything large in areas the maintainer's projects use

Keep it to a handful of lines. If the sync brought hundreds of pull requests,
summarize by area.

If anything about the toolchain or procedure had to change to make the sync
work, update `CLAUDE.md`, the two docs in `fork\` and this skill in the same
session so the next sync starts from the truth. The fork is public, so keep
absolute paths and personal details out of anything committed.
