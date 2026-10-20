# Build and run — commands, prerequisites, artifacts

All paths are under the recovery workspace
`C:\Users\choicky\worktmp\fusion-windows\official-settings-ui\spike\` (`$SPIKE` below).
Nothing here is committed to Git; this file records how the result was produced.

## Prerequisites

- Visual Studio 18 Community, MSVC 14.51.36231 + Windows SDK 10.0.26100.0 (present on the machine).
- Qt 6.8.3 `win64_msvc2022_64`, `qtbase` archive only, installed task-locally with
  `aqtinstall` 3.3.0: `python -m aqt install-qt windows desktop 6.8.3 win64_msvc2022_64
  --archives qtbase --outputdir $SPIKE/qt`.
- ECM 6.10.0 from `download.kde.org/stable/frameworks/6.10/extra-cmake-modules-6.10.0.tar.xz`,
  extracted (the local `tar` lacks xz, Python `tarfile` was used), configured and installed to
  `$SPIKE/deps/ecm-install` (build-time only).
- The pinned runtime, reused and not rebuilt:
  `C:\Users\choicky\worktmp\fusion-windows\runtime-wp3-20261018\runtime-release`.
- Pinned upstream checkouts copied into `$SPIKE/upstream/` and patched (see `patch-inventory.md`).

Machine-specific notes that the commands below compensate for: `VsDevCmd.bat` is unusable in this
session (it invokes `reg.exe`, blocked), so `INCLUDE`/`LIB`/`PATH` are set explicitly in
`$SPIKE/env.sh`; bare `cl`, `rc`, `mt`, `ninja` and `cmake` are not resolved through `PATH`, so they
are named outright; this machine's `System32` has no `api-ms-win-crt-*` forwarders, so the UCRT
redistributable directory must be on `PATH` when running anything linked against the dynamic CRT.

## Environment

```bash
source $SPIKE/env.sh   # sets INCLUDE, LIB, PATH (MSVC, SDK, CMake, Ninja, Qt, UCRT redist),
                       # CC/CXX, CMAKE/NINJA, QTPREFIX, and CMAKE_ARGS (Ninja + Release + toolchain)
```

`CMAKE_ARGS` is: `-G Ninja -DCMAKE_MAKE_PROGRAM=<ninja> -DCMAKE_BUILD_TYPE=Release
-DCMAKE_PREFIX_PATH=<qt>;<ecm> -DECM_DIR=<ecm>/share/ECM/cmake -DCMAKE_RC_COMPILER=<rc>
-DCMAKE_MT=<mt> -DCMAKE_CXX_COMPILER=<cl>`.

## Build

```bash
# 1. QtDBus transport probe
"$CMAKE" -S "$SPIKE/src/probe" -B "$SPIKE/build/probe" "${CMAKE_ARGS[@]}"
"$NINJA" -C "$SPIKE/build/probe"

# 2. Experimental settings window: fcitx5-qt dbusaddons+widgetsaddons, configlib,
#    configwidgetslib, and the window itself (one CMake project)
"$CMAKE" -S "$SPIKE/src/settings-app" -B "$SPIKE/build/ui" "${CMAKE_ARGS[@]}" \
  -DFUSION_FCITX5_ROOT="C:/Users/choicky/worktmp/fusion-windows/runtime-wp3-20261018/runtime-release"
"$NINJA" -C "$SPIKE/build/ui"

# 3. Isolated engine service and pipe CLI (same pattern)
"$CMAKE" -S "$SPIKE/src/engine-service" -B "$SPIKE/build/engine" "${CMAKE_ARGS[@]}" \
  -DFUSION_FCITX5_ROOT="<runtime-release>"
"$NINJA" -C "$SPIKE/build/engine"
"$CMAKE" -S "$SPIKE/src/pipe-cli" -B "$SPIKE/build/cli" "${CMAKE_ARGS[@]}"
"$NINJA" -C "$SPIKE/build/cli"
```

Build outputs (not committed; recorded here as build evidence):

| Artifact | Bytes | SHA-256 |
| --- | --- | --- |
| `fusion-settings-spike.exe` | 317952 | `94008a1f42e7fead98ef93531e49aecf664f597889337f866a96e9eef953663e` |
| `Fcitx5Qt6DBusAddons.dll` | 337408 | `800d27f4503a17611a4327c6938d742a6ecd12ff1c10e8dada23a0711902b9ac` |
| `Fcitx5Qt6WidgetsAddons.dll` | 80384 | `9b98d3451b4729223e9c9a561354042f4c3853fa3af54577e3dac85415f8fcbb` |
| `fusion-settings-engine.exe` | 978432 | `4f6d4fc8fd1a0e5cc4ec49fb3a85658a426e4209743309eb9d9afca28503fd15` |
| `pipe-cli.exe` | 45568 | `74d61b4c675e04661a5ad3692c141854981b49b8e3a793946af440d5947c1b9e` |
| `seam-probe.exe` (transport probe) | 62464 | `bccf050bffcb14d8698e4ba75cd379658eabfbcf850c6c5ada59dcda2d64df28` |

## Run

The engine, the CLI and the window are always launched through PowerShell: Git Bash does not use
the executable's directory as a DLL search path, and `Start-Process` fails in this session
(duplicate `http_proxy`/`HTTP_PROXY` keys), so the runners create processes with
`System.Diagnostics.ProcessStartInfo`.

```powershell
# Full UI round trip on a fresh disposable profile (baseline read -> UI apply -> re-read ->
# orderly restart -> no-change pass). Every run redirects APPDATA/LOCALAPPDATA/USERPROFILE.
powershell -File $SPIKE\run-ui.ps1 -Profile p9 -PipeName fusion-settings-ui-1 -Action All

# Transport/semantics only, without the widget layer
powershell -File $SPIKE\run-spike.ps1 -Profile p6 -PipeName fusion-settings-spike-1 -Action Read -Fresh -DumpSchema
powershell -File $SPIKE\run-spike.ps1 -Profile p7 -PipeName fusion-settings-spike-1 -Action Apply -PageSize 9
```

Each run uses a new `-Profile` name: this session's safe-delete wrapper refuses recursive deletion,
so profiles are never reused or deleted. Transcript: `ui-roundtrip.log` in this directory.
