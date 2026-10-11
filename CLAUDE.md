# Godot fork

A personal fork of Godot Engine. The goal is to stay current with upstream
development while carrying customizations for the maintainer's own projects.
"The maintainer" in these files means the person you are working with.
Typical requests: sync with upstream, merge into the custom branch, rebuild,
build export templates, fix a build that broke after a sync.

## Layout

Paths in this file and in `fork\` are relative to the repository root.

```
CLAUDE.md                                this file
fork\walkthrough.md                      toolchain setup and build reference
fork\upstream-sync.md                    sync and merge procedure
.claude\skills\sync-upstream\SKILL.md    the /sync-upstream skill
bin\                                     build output (git-ignored)
```

These four files exist only on the `custom` branch. Upstream does not track
or ignore any of these paths, so they never conflict in a merge.

Keep `custom` checked out. Checking out `master`, or a branch made from it,
removes these files from the working tree, this one included, and a session
started in that state has no instructions and no skill.

## Remotes and branches

- `origin` = `https://github.com/coreyx/godot.git` (the fork, public)
- `upstream` = `https://github.com/godotengine/godot.git`
- `master` mirrors `upstream/master`. Never commit to it and do not check it
  out. Update it in place with `git fetch . upstream/master:master`, which
  only fast-forwards and refuses if `master` has diverged.
- `custom` holds these docs, the skill, and the engine customizations. It is
  updated with `git merge master`.
- `rerere.enabled=true` is set in the clone.

For any sync request, invoke the `sync-upstream` skill. Modes: no argument
for a full sync without pushing, `check` to only report how far behind the
fork is, `push` to sync and then push. If the skill is missing from the skill
list, read `.claude\skills\sync-upstream\SKILL.md` and follow it by hand.

`fork\upstream-sync.md` is the human-facing version of the same procedure
and holds the recovery steps the skill points to. Keep the two in step: a
change to the procedure goes in both.

The harness blocks Claude from writing under `.claude\`. To change the skill,
write the new `SKILL.md` somewhere outside the repository, ask the maintainer
to move it into place, then commit it.

Untested as of 2026-10-10: the skill's fast-forward with real new upstream
commits, a `custom` merge that brings in upstream changes, conflict handling,
and the failed-build branches. The individual git commands have been tested
on throwaway branches, and the read-only steps have been run for real. Treat
the first real sync as the test, and fix the skill and docs if a step turns
out wrong.

## This machine: things that will trip you up

- Use `py -m SCons`, never `scons`. pip's Scripts directory is not on PATH.
- `py` and `python` both work (the maintainer disabled the Microsoft Store
  `python` stub on 2026-10-10). The docs use `py`; keep doing so.
- Run builds and the `install_*.py` scripts with the PowerShell tool, not
  Bash. Git Bash sets `MSYSTEM`, and Godot's scripts then look for build
  dependencies in `bin\build_deps` instead of `%LOCALAPPDATA%\Godot\build_deps`
  and detect the architecture differently. Upstream's docs also say MSVC
  builds must not run from MSYS2/MinGW shells.
- A full editor build takes about 4.5 minutes here (32 cores); a no-change
  rebuild about 40 seconds. Run full builds in the background with output
  redirected to a log file in the scratchpad, then read the log tail. To
  block until one finishes, `Wait-Process -Id <scons pid> -Timeout 570`.
- `WARNING: msgfmt not found` during a build is harmless.
- Do not pipe the editor exe into `Select-Object -First N` when smoke
  testing. It ends the pipeline early, kills the process and reports exit -1.
  Capture the output into a variable and read `$LASTEXITCODE` straight after.
- `Start-Sleep` polling is blocked in this harness. Use `run_in_background`
  and wait for the notification.
- The session is not elevated. Anything needing admin (VS installer changes,
  Defender exclusions) raises a UAC prompt the maintainer must accept, or
  must be run by the maintainer.

## Toolchain (installed 2026-10-10)

| Tool | Version | Notes |
|---|---|---|
| VS 2022 Build Tools | 17.14.41 | C++ workload; MSVC 14.44.35207 |
| Windows SDK | 10.0.26100.0 | installed with the workload |
| Python | 3.14.7 | `py` launcher |
| SCons | 4.11.1 | `py -m SCons` |
| D3D12 / ANGLE / AccessKit libs | see walkthrough | `%LOCALAPPDATA%\Godot\build_deps` |

## Common commands

From the repository root:

```powershell
# Build the editor (incremental)
py -m SCons platform=windows

# Smoke test the result (both should exit 0)
.\bin\godot.windows.editor.x86_64.console.exe --version
.\bin\godot.windows.editor.x86_64.console.exe --headless --quit

# Debuggable engine build, for working on engine code
py -m SCons platform=windows dev_build=yes

# Export templates
py -m SCons platform=windows target=template_debug
py -m SCons platform=windows target=template_release

# Clean
py -m SCons platform=windows --clean

# Refresh build dependencies after upstream bumps their versions
py misc/scripts/install_d3d12_sdk_windows.py
py misc/scripts/install_accesskit.py
py misc/scripts/install_angle.py

# Update the mirror and bring custom forward, without leaving custom
git fetch upstream
git fetch . upstream/master:master
git merge --no-edit master
```

## Rules

- Never commit to `master`. Commit customizations to `custom`.
- Do not push, force-push, or open pull requests unless the maintainer asks
  in that session. Local fetches, merges, commits and builds are fine.
- The fork is public. Keep absolute paths, usernames and anything personal
  out of committed files; use paths relative to the repository root.
- Before a sync, `git status` must be clean. Stop and report if it is not.
- After a sync or merge, rebuild and run the smoke test before calling it
  done. Report the upstream commit range that came in.
- If a merge conflicts in a way that needs a judgment call about a
  customization, show the conflict and ask instead of guessing.
- When the toolchain, branch layout or procedure changes, update this file,
  the two docs in `fork\` and the skill in the same session.
- Verify state with git before relying on anything written here. This file
  records how things were set up, not what they look like today.
