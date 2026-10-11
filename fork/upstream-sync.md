# Keeping the fork up to date with upstream Godot

How this fork tracks `godotengine/godot` while carrying personal changes.
Toolchain setup and build details are in [walkthrough.md](walkthrough.md).

All commands run in PowerShell from the repository root, with the `custom`
branch checked out.

## The quick way: `/sync-upstream`

Claude Code has a skill that runs this whole procedure. Start Claude Code in
the repository root (the fork's own folder, the one containing `SConstruct`)
with `custom` checked out, and type one of:

| Command | What it does |
|---|---|
| `/sync-upstream check` | Fetches upstream and reports how far behind the fork is and what is waiting. Changes nothing else |
| `/sync-upstream` | Full sync: fast-forwards `master`, merges it into `custom`, rebuilds the editor, smoke tests it, reports what came in. Does not push |
| `/sync-upstream push` | Full sync, then pushes `master` and `custom` to the GitHub fork |

Asking in plain words works too, for example "sync the fork with upstream" or
"how far behind upstream am I?".

**It stops and asks you** instead of guessing when:

- the clone has uncommitted work, or a merge is already in progress
- a branch other than `custom` is checked out
- something was committed directly to `master`
- upstream rewrote or deleted code that one of your customizations lived in

**It will not** push unless asked, push a branch that fails to build, or
stash, discard or reset anything on its own.

**What you get back** is a short report:

```
Result: synced | already up to date | stopped: <reason>
Upstream: <old hash>..<new hash>, <N> pull requests, <date range>
Custom branch: merged cleanly | merged with conflicts in <files> | nothing to merge
Build: passed in <time> | failed: <one line> | skipped: <why>
Smoke test: <version string>, headless start OK
Pushed: yes | no

Worth knowing:
- <the few upstream changes that matter to you>
```

The skill is a plain Markdown file at `.claude\skills\sync-upstream\SKILL.md`;
edit it to change how syncs behave. The rest of this document is the same
procedure by hand, plus the recovery steps the skill refers to. If you change
one, change the other.

As of 2026-10-10 the skill's read-only steps have been run against the real
clone, and its git commands have been tested on throwaway branches. A sync
that brings in real upstream commits, a conflicting merge and a failed build
have not happened yet. Read the report from the first real sync with that in
mind.

## How the fork is arranged

| Thing | Value |
|---|---|
| `origin` | `https://github.com/coreyx/godot.git` (the fork) |
| `upstream` | `https://github.com/godotengine/godot.git` |
| `master` | Exact mirror of `upstream/master`. Never commit to it or check it out |
| `custom` | Everything that is yours: engine changes, these docs, the Claude Code files. Stays checked out |

Keeping `master` as a clean mirror means updating it is always a fast-forward
that cannot conflict, and `git diff master...custom` always shows exactly what
has been customized. Conflicts can only happen in one place: merging `master`
into `custom`.

`CLAUDE.md`, the `fork\` folder and the skill exist only on `custom`. That is
why the procedure below updates `master` without checking it out: switching
to `master` would take those files off disk until you switched back.

## Routine sync

Upstream `master` gets dozens of commits a day. Syncing weekly, or before
starting any new customization, keeps each merge small.

**1. Start clean, on `custom`.**

```powershell
git status
```

Commit or stash anything outstanding first.

**2. Update the mirror.**

```powershell
git fetch upstream
git fetch . upstream/master:master
```

The second command moves the local `master` branch forward to match
upstream without touching the working tree. It only fast-forwards; if
`master` has commits of its own it prints `[rejected] (non-fast-forward)` and
changes nothing (see "If the fast-forward is rejected").

**3. Bring `custom` forward.**

```powershell
git merge master
```

Resolve any conflicts (next section).

**4. Rebuild and test.**

```powershell
py -m SCons platform=windows
.\bin\godot.windows.editor.x86_64.console.exe --version
.\bin\godot.windows.editor.x86_64.console.exe --headless --quit
```

The build is incremental, so only what changed gets recompiled.

**5. Publish.**

```powershell
git push origin master custom
```

Pushing starts upstream's CI workflow on the fork, because GitHub Actions is
enabled there and the workflow runs on every push. It costs nothing on a
public repository. To stop it, disable Actions under the fork's Settings >
Actions > General.

## Making a customization

Commit on `custom`:

```powershell
git switch custom
# ...make changes...
git commit -am "Describe the change and why"
```

## Resolving merge conflicts

```powershell
git diff --name-only --diff-filter=U   # files still in conflict
git merge --abort                      # back out entirely and try later
```

For each conflicted file, keep upstream's new structure and re-apply the
customization on top of it. Taking the old customized version wholesale tends
to silently revert upstream fixes. Then:

```powershell
git add <file>
git merge --continue
```

`rerere` is enabled in this clone (`git config rerere.enabled true`), so git
records each resolution and replays it if the same conflict comes back.

### Making conflicts rarer

- Put new functionality in its own module instead of editing engine files.
  Modules can live outside the tree entirely and be pulled in at build time
  with `custom_modules=<path to a folder of modules>`, which makes them
  immune to upstream merges.
- When an engine file must be edited, keep the edit small and commit it
  separately with a message that says why, so a future conflict is easy to
  re-apply.
- Do not reformat or reorder upstream code.

## After a sync: things that break

| Symptom | Fix |
|---|---|
| SCons says D3D12, AccessKit or ANGLE libraries are missing or the wrong version | Re-run the matching `py misc/scripts/install_*.py` (see walkthrough) |
| Odd compile or link errors after a large jump | Clean, then rebuild: `py -m SCons platform=windows --clean` |
| SCons or Python version too old | `py -m pip install --upgrade scons` |
| MSVC not detected, or compiler internal errors | Update VS Build Tools: `winget upgrade --id Microsoft.VisualStudio.2022.BuildTools` |
| A project opens with import or script errors | `master` is a dev branch. Commit the project before opening it with a newer build, so the upgrade can be reverted |

## If the fast-forward is rejected

Something was committed to `master` directly. Keep those commits on another
branch before moving `master`, so nothing is lost:

```powershell
git log --oneline upstream/master..master   # the stray commits
git branch rescue-master master             # keep them
git branch -f master upstream/master        # moves master; rescue-master still has the commits
git push --force-with-lease origin master   # only if the stray commits were pushed
```

Then cherry-pick what you want from `rescue-master` onto `custom`.

## Tracking a stable release instead of `master`

`upstream/master` is the development branch (4.8-dev at the time of writing).
For a project that needs to ship, base a second customization branch on a
stable branch:

```powershell
git fetch upstream --tags
git switch -c custom-4.7 upstream/4.7
git merge custom        # or cherry-pick only the commits you want
```

and sync it with `git merge upstream/4.7`. Stable branches only receive bug
fixes, so those merges are small. Release tags look like `4.7.2-stable`.

## Sending a change back to upstream

Branch from `master`, not `custom`, so the pull request carries only that one
change and none of the fork's own files:

```powershell
git switch -c fix-something master
# ...commit...
git push -u origin fix-something
git switch custom
```

Then open the pull request against `godotengine/godot`. While that branch is
checked out, `CLAUDE.md`, `fork\` and the skill are not on disk; they come
back with `git switch custom`.
