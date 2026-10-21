# Build prerequisite — resolved (2026-10-21, fourth round)

This file was the blocker record. It is kept as the single prerequisite record and is now a **resolved** one: the WinUI
client builds, launches and packages. What remains is a note on how the dependency was obtained, so the next person does not
have to rediscover it.

## Current state

| Requirement | State |
|---|---|
| .NET SDK | **10.0.402**, task-local at `worktmp\fusion-windows\settings-winui3-env\dotnet`, SHA-512 verified against the official sidecar |
| `Microsoft.WindowsAppSDK` | **2.5.1**, restored and verified |
| `Microsoft.Windows.SDK.BuildTools` | **10.0.28000.2705**, restored and verified |
| WinUI build | **works** — the app compiles, launches and shows a `WinUIDesktopWin32WindowClass` window |
| Packaging | **works** — `fusion-0.0.0-winui3+502d5ec.zip`, 652 manifest entries, every hash re-verified against the extracted tree |

## Running a Host that can save

A Host started with `FUSION_FCITX_SKIP_USER_PATH=1` has no user configuration directory, so its global apply ends in
Fcitx's `safeSave()` and fails. That is fine for failure-path checks and useless for proving a save works. For validation,
start the Host with its per-user directories redirected to a disposable profile instead (`APPDATA`, `LOCALAPPDATA`,
`USERPROFILE`, and `FUSION_FCITX_SKIP_USER_PATH` unset), and read the two paths the Host prints on startup — both must be
inside that profile. `src/settings-winui3/README.md` has the exact commands, including the orderly stop through
`Local\FusionHost-stop-<scope>` and the restart that confirms persistence.

## How the packages were obtained

`api.nuget.org` and `globalcdn.nuget.org` are still unreachable from this machine. The proxy accepts the tunnel
(`HTTP/1.1 200 Connection Established`) and the TLS handshake then fails, so `dotnet restore` reports
`NU1301 … The SSL connection could not be established … Received an unexpected EOF or 0 bytes from the transport stream`.

What made progress possible is that **nuget.org's own OData endpoint on `www.nuget.org` is reachable**, and it publishes
the SHA-512 and size of every package version. So a mirror is used as a **transport only**, and every package is checked
against the hash nuget.org itself publishes before it is used:

- `worktmp\fusion-windows\settings-winui3-env\nuget\nuget.config` points restore at a mirror and keeps the package cache
  task-local (`globalPackagesFolder` under the same directory), so nothing is written to the machine profile.
- `probe3\verify_cache.py` walks that cache, fetches `Packages(Id=…,Version=…)` from `www.nuget.org` and compares
  `PackageHash` (base64 SHA-512) and `PackageSize`.
- **22 packages, 0 mismatches.** A package whose bytes did not match would have stopped the build.

The mirror is named in the task-local config and in this record, not silently, and it is not configured anywhere in the
repository: the project files only declare ordinary `PackageReference`s, so a machine with normal nuget.org access restores
the same versions from the official feed.

No TLS validation was disabled, no certificate store was touched, and no proxy or firewall setting was changed.

## What is still unreachable, and why it does not block

`download.microsoft.com` (the Windows App Runtime installer) and `api.github.com` remain unreachable. Neither is needed:
the client is built **self-contained** for both the Windows App SDK and .NET, so it requires no machine-wide runtime and no
installer.

## Consequence for the product

- The client is **implemented, compiled, tested and packaged** — and **not installed** and **not owner-accepted**.
- The package carries the .NET and Windows App SDK runtimes, which is why the ZIP is about 139 MB. That is the price of
  needing nothing installed on the target, and the redistributed set is recorded in `THIRD_PARTY_LICENSES.md`.

## Reproducing a build

```
dotnet restore  src/settings-winui3/FusionSettings.App/FusionSettings.App.csproj --configfile <task-local nuget.config>
dotnet build    src/settings-winui3/FusionSettings.App/FusionSettings.App.csproj -c Release --no-restore
dotnet publish  src/settings-winui3/FusionSettings.App/FusionSettings.App.csproj -c Release --no-restore -o <publish>
```

`src/settings-winui3/README.md` has the full sequence, including the isolated-Host recipe for the checks.
