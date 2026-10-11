# Godot fork: Windows build setup walkthrough

What was done on 2026-10-10 to make this fork compile on Windows, and how to
repeat it. Based on the official guide,
[Compiling for Windows](https://docs.godotengine.org/en/latest/engine_details/development/compiling/compiling_for_windows.html).

Related files, all on the `custom` branch:

- [upstream-sync.md](upstream-sync.md): keeping the fork current with upstream
- [../CLAUDE.md](../CLAUDE.md): notes Claude Code loads when working here
- `.claude\skills\sync-upstream\SKILL.md`: the `/sync-upstream` skill

## Layout

Paths are relative to the repository root.

```
CLAUDE.md                                notes for Claude Code
fork\walkthrough.md                      this file
fork\upstream-sync.md                    sync procedure
.claude\skills\sync-upstream\SKILL.md    the /sync-upstream skill
bin\                                     build output (git-ignored)
```

The first four exist only on the `custom` branch. `master` is an untouched
mirror of upstream.

## Starting point

| Item | State before |
|---|---|
| Fork clone | Present, `master` identical to `upstream/master` (4.8-dev, `232e6b14ab`) |
| Remotes | `origin` (coreyx/godot) and `upstream` (godotengine/godot) already configured |
| Git | Installed |
| Python | 3.14.7 installed, reachable as `py` |
| C++ compiler | None (no Visual Studio, Windows SDK or MinGW) |
| SCons | Not installed |

## What was done

### 1. Installed Visual Studio 2022 Build Tools

The official guide recommends MSVC from Visual Studio 2022. The Build Tools
edition gives the compiler and Windows SDK without the IDE.

```powershell
winget install --id Microsoft.VisualStudio.2022.BuildTools --source winget `
  --accept-package-agreements --accept-source-agreements `
  --override "--passive --wait --norestart --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
```

This raises one UAC prompt. `--includeRecommended` is what pulls in the MSVC
x64/x86 toolset and the Windows 11 SDK. No reboot was needed.

Result: Build Tools 17.14.41, MSVC 14.44.35207, Windows SDK 10.0.26100.0.

To verify:

```powershell
& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -products * `
  -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property installationPath
```

It should print the Build Tools path. There is no need to open a "Developer
Command Prompt"; SCons finds MSVC on its own from a normal PowerShell window.

### 2. Installed SCons

```powershell
py -m pip install --upgrade scons
py -m SCons --version
```

Result: SCons 4.11.1 (Godot needs 4.4 or newer, and Python 3.9 or newer).

### 3. Downloaded the optional Windows build dependencies

Godot's Windows build uses prebuilt libraries for three features. Without
them the build either stops or drops the feature. The repo ships scripts that
fetch them. From the repository root:

```powershell
py misc/scripts/install_d3d12_sdk_windows.py   # Direct3D 12 renderer
py misc/scripts/install_accesskit.py           # screen reader support
py misc/scripts/install_angle.py               # OpenGL ES over Direct3D 11
```

They install to `%LOCALAPPDATA%\Godot\build_deps`, outside the repo, so they
survive cleans and branch switches.

| Dependency | Version fetched |
|---|---|
| Mesa NIR | 25.3.1-3 |
| WinPixEventRuntime | 1.0.240308001 |
| DirectX 12 Agility SDK | 1.618.5 |
| AccessKit | 0.23.1 |
| ANGLE | chromium/7219 |

The versions are pinned inside the scripts. When upstream bumps one, the build
will say so and the fix is to re-run the matching script.

### 4. Built the editor

From the repository root:

```powershell
py -m SCons platform=windows
```

SCons reported `Using Visual Studio 14.3 with Windows SDK 10.0.26100.0`,
target `editor`, architecture `x86_64`, 31 parallel jobs.

Measured on this machine (32 cores, 94 GB RAM):

| Build | Time |
|---|---|
| Full editor build from scratch | 4 min 21 s |
| Rebuild with nothing changed | about 40 s |

### 5. Verified the result

```powershell
.\bin\godot.windows.editor.x86_64.console.exe --version
# 4.8.dev.custom_build.232e6b14a

.\bin\godot.windows.editor.x86_64.console.exe --headless --quit
# Godot Engine v4.8.dev.custom_build.232e6b14a ... exits with code 0
```

`--help` lists the rendering drivers `vulkan`, `d3d12`, `opengl3` and
`opengl3_angle`, which confirms the Direct3D 12 and ANGLE dependencies from
step 3 were picked up. The editor GUI itself was not opened during setup; to
launch it, run `.\bin\godot.windows.editor.x86_64.exe`.

### 6. Configured git

- `git config rerere.enabled true` in the clone, so git remembers merge
  conflict resolutions (see upstream-sync.md).
- `git fetch upstream` to pull all upstream branches and tags.

### 7. Added the `/sync-upstream` skill

A Claude Code skill that automates the routine in upstream-sync.md: fetch
upstream, fast-forward `master`, merge into `custom`, rebuild, smoke test and
report. It is used by starting Claude Code in the repository root and typing
`/sync-upstream`, `/sync-upstream check` or `/sync-upstream push`. Usage and
guardrails are described at the top of [upstream-sync.md](upstream-sync.md).

Claude writes the skill file but is not permitted to place anything under
`.claude\` itself, so each new version is moved into place by hand.

### 8. Created the `custom` branch and moved the docs into it

These docs, `CLAUDE.md` and the skill were first written in the folder that
contains the clone, outside git. They were then moved into the fork on a new
`custom` branch so they are versioned and travel with the clone. Absolute
paths were replaced with paths relative to the repository root, because the
fork is public.

`master` was not touched and still matches upstream exactly.

## Quirks on this machine

**`python` was broken during setup, and is fixed now.** Bare `python` used to
resolve to the Microsoft Store stub in `%LOCALAPPDATA%\Microsoft\WindowsApps`,
which sat ahead of the real Python on PATH and printed "Python was not found".
The stub was turned off later the same day (Settings > Apps > Advanced app
settings > App execution aliases), so `python` and `py` now both work. The
commands in these docs use `py`, which works either way.

**Use `py -m SCons`, not `scons`.** pip put `scons.exe` in
`%LOCALAPPDATA%\Python\pythoncore-3.14-64\Scripts`, which is not on PATH.
Running it as a module avoids touching PATH. Wherever the official docs say
`scons ...`, type `py -m SCons ...`.

## Everyday builds

All from the repository root, in PowerShell or cmd (not Git Bash).

| Goal | Command |
|---|---|
| Editor, optimized (the default) | `py -m SCons platform=windows` |
| Editor for debugging engine code | `py -m SCons platform=windows dev_build=yes` |
| Debug export template | `py -m SCons platform=windows target=template_debug` |
| Release export template | `py -m SCons platform=windows target=template_release` |
| Editor with unit tests | `py -m SCons platform=windows tests=yes`, then run the exe with `--test` |
| Remove build output | `py -m SCons platform=windows --clean` |
| Limit CPU use | add `-j 16` |
| `compile_commands.json` for clangd | add `compiledb=yes` |
| Visual Studio solution | add `vsproj=yes` |
| List every option | `py -m SCons --help` |

Builds are incremental. After the first one, only changed files recompile.

### Output

Everything lands in `bin\`:

- `godot.windows.editor.x86_64.exe`: the editor
- `godot.windows.editor.x86_64.console.exe`: same, attached to the terminal
  so output is visible; use this one from scripts
- `dev_build=yes` adds `.dev` to the name, so both builds can coexist
- Templates are named `godot.windows.template_debug.x86_64.exe` and
  `godot.windows.template_release.x86_64.exe`

### Using custom export templates

A project exported with this fork's editor needs templates built from the same
source. Build both template targets, then in the project's export preset turn
on Advanced Options and point Custom Template > Debug and Release at the two
files in `bin\`.

### Persistent build options

To avoid retyping options, create `custom.py` in the repository root (already
git-ignored by upstream):

```python
platform = "windows"
compiledb = "yes"
```

After that, plain `py -m SCons` picks them up.

### Keeping settings separate from a stock Godot install

Create an empty file named `._sc_` in `bin\`. The editor then stores its
settings beside the exe instead of in `%APPDATA%\Godot`, so this build cannot
disturb an official Godot install on the same machine.

## Not done: Windows Defender exclusion

The official guide recommends excluding the source folder from Defender
because real-time scanning slows builds noticeably. It needs administrator
rights and loosens a security setting, so it was left as a decision for the
maintainer. From an elevated PowerShell in the repository root:

```powershell
Add-MpPreference -ExclusionPath (Get-Location).Path
```

## Repeating this on a fresh machine

This relies on `custom` having been pushed to the fork.

```powershell
# 1. Tools
winget install --id Git.Git --source winget
winget install --id Python.PythonInstallManager --source winget   # or any Python 3.9+
winget install --id Microsoft.VisualStudio.2022.BuildTools --source winget `
  --accept-package-agreements --accept-source-agreements `
  --override "--passive --wait --norestart --add Microsoft.VisualStudio.Workload.VCTools --includeRecommended"
py -m pip install --upgrade scons

# 2. Source
git clone https://github.com/coreyx/godot.git
cd godot
git switch custom
git remote add upstream https://github.com/godotengine/godot.git
git fetch upstream
git config rerere.enabled true

# 3. Build dependencies
py misc/scripts/install_d3d12_sdk_windows.py
py misc/scripts/install_accesskit.py
py misc/scripts/install_angle.py

# 4. Build
py -m SCons platform=windows
```

## Keeping the toolchain current

```powershell
winget upgrade --id Microsoft.VisualStudio.2022.BuildTools   # compiler and SDK
py -m pip install --upgrade scons                            # build system
```

Re-run the three `install_*.py` scripts whenever the build reports a missing
or outdated dependency.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| "Python was not found" | The Microsoft Store stub is back or was never disabled. Use `py`, or turn the stub off under App execution aliases |
| `scons` is not recognized | Not on PATH; use `py -m SCons` |
| SCons cannot find a compiler | Build Tools missing the C++ workload. Run the Visual Studio Installer, choose Modify, tick "Desktop development with C++". Also try `py -m pip install --upgrade scons` |
| Build complains about D3D12, AccessKit or ANGLE | Re-run the matching `install_*.py`. To build without the feature, pass `d3d12=no`, `accesskit=no` or `angle=no` |
| Dependencies not found when building from Git Bash | Git Bash sets `MSYSTEM`, which changes where Godot looks. Build from PowerShell |
| Strange errors after a large upstream merge | `py -m SCons platform=windows --clean`, then rebuild |
| `WARNING: msgfmt not found, using .po files instead of .mo` | Harmless. gettext is not installed, so editor translations are embedded in a less compact form. Ignore it |
| `production=yes` runs out of memory | Link-time optimization is memory hungry. Add `lto=none` |
| `CLAUDE.md`, `fork\` or the skill have disappeared | A branch other than `custom` is checked out. `git switch custom` |
