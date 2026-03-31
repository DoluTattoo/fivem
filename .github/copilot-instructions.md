# FiveM Local Build — Copilot Context

## Repository Setup

This is a fork of `citizenfx/fivem` configured to build locally on Windows.
- **Branch `my-custom-build`** contains all local build adaptations on top of upstream `master`
- **Remote `origin`** → citizenfx/fivem (upstream)
- **Remote `me`** → DoluTattoo/fivem (personal fork)

## Known Issues & Workarounds

### Git LFS Budget Exceeded
The upstream citizenfx/fivem repo has exceeded its Git LFS bandwidth quota. All `git lfs pull` calls fail.
- **Affected files**: `vendor/libnode/bin/` (libnode22.dll/lib/pdb, libuv.dll/lib/pdb)
- **Workaround**: Binaries were downloaded manually from `https://github.com/citizenfx/libnode/releases/download/v22.22.0/`
- The old download logic existed in `code/vendor/libnode.lua` before commit `5a150e22b` moved files to LFS

### Git Symlinks (core.symlinks)
The repo uses 14 git symlinks (mode 120000) for shared code between components. On Windows, these only work if cloned with `git clone -c core.symlinks=true` (requires Developer Mode or admin).
- **Our fix**: Replaced broken symlink text files with NTFS directory junctions (`mklink /J`), which don't need elevation
- **Affected paths**:
  - `code/client/clrcore-v2/Client/FiveM/v1` → `../../../clrcore/External/`
  - `code/client/clrcore-v2/Math` → `../clrcore/Math`
  - `code/client/clrcore-v2/Server` → `../clrcore/Server`
  - `code/components/citizen-scripting-v8-v12.4/{include,src}` → `../citizen-scripting-v8/`
  - `code/components/citizen-scripting-v8client/{include,src}` → `../citizen-scripting-v8/`
  - `code/components/citizen-scripting-v8node/{include,src}` → `../citizen-scripting-v8/`
  - `code/components/citizen-server-state-fivesv/{include,src}` → `../citizen-server-state/`
  - `code/components/citizen-server-state-rdr3sv/{include,src}` → `../citizen-server-state/`
  - `code/vendor/breakpad/src/third_party/lss` → `../../../../../vendor/lss`
- **If symlinks break again**: After `fxd gen -game five`, the .csproj/.vcxproj won't include files from symlinked dirs → CS0246 errors (missing Ped, Vehicle, etc.) and LNK1181 errors

### Windows SDK Version
- `code/premake5.lua` line 179: Changed `systemversion` from `'10.0.22000.0'` to `'10.0.22621.0'`
- SDK 10.0.22000.0 is no longer available in the VS2022 installer catalog
- Must match an SDK version actually installed on the machine

### Visual Studio Version Selection
- `code/tools/fxd/.helpers.psm1` line 26: Changed vswhere from `-prerelease -latest` to `-version "[17.0,18.0)"`
- This machine has both VS2022 (v17) and VS2026 Preview (v18). The original `-latest` flag picked VS2026 which doesn't have ATL available
- If only VS2022 is installed, this change is harmless

## Build Pipeline

The full build sequence is:
```powershell
# 1. Download CEF (Chromium Embedded Framework)
.\fxd.ps1 get-chrome

# 2. Generate native bindings (requires MSYS2 at C:\msys64 with make/diffutils)
.\prebuild.cmd

# 3. Generate VS2022 solution via Premake5
.\fxd.ps1 gen -game five

# 4. Build
.\fxd.ps1 build -game five
```

**Output directory**: `code/bin/five/debug/` (FiveM.exe + ~200 DLLs, ~2.85 GB total)

### Incremental builds
After editing source files, only step 4 is needed. Steps 1-3 are only needed after:
- First clone / clean checkout
- Changes to premake5.lua or .lua build scripts
- Changes to native definitions

## Build Prerequisites

- **Visual Studio 2022** with: C++ Desktop workload, ATL (VC.ATL), .NET 4.6 targeting pack, Windows 11 SDK 10.0.22621.0
- **Python 3** with packages: Jinja2, MarkupSafe, ply, six, setuptools
- **Node.js + Yarn** (yarn 1.x via `npm install -g yarn`)
- **MSYS2** at `C:\msys64` with `make` and `diffutils` (`pacman -S make diffutils`)
- **Git submodules**: All 151 must be initialized (`git submodule update --init --jobs=16`)

## Architecture Notes

- **Premake5** generates VS2022 solutions (binary: `code/tools/ci/premake5.exe`)
- **fxd.ps1** is the build orchestrator, dispatches to scripts in `code/tools/fxd/`
- **vswhere.exe** bundled at `code/tools/ci/vswhere.exe` — selects VS installation
- **NuGet.exe** bundled at `code/tools/ci/nuget.exe`
- The `vendor/node` submodule (Node 16.9.1) is NOT what the build uses — the build links against prebuilt libnode22 (Node 22) from `vendor/libnode/bin/`
- Native definitions are downloaded from `https://static.cfx.re/natives/` during prebuild

## Key Files

| File | Purpose |
|------|---------|
| `code/premake5.lua` | Main build config, all projects, SDK version |
| `code/tools/fxd/.helpers.psm1` | VS initialization, vswhere invocation |
| `code/tools/fxd/build.ps1` | Build entry point (NuGet restore + MSBuild) |
| `code/tools/fxd/gen.ps1` | Solution generation (premake5 vs2022) |
| `code/vendor/libnode.lua` | libnode22 binary copy rules |
| `code/vendor/libuv.lua` | libuv binary copy rules |
| `prebuild.cmd` | Native binding generation pipeline |
| `.gitattributes` | LFS tracking rules for vendor/libnode/bin/ |
