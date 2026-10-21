# Build prerequisite — what this machine can and cannot build (2026-10-21, updated)

Originally recorded when no .NET SDK was present. The SDK half is now **resolved**; the Windows App SDK half is still
blocked. The file is kept as the single prerequisite record rather than being replaced by a second one.

## Current state

| Requirement | State | Evidence |
|---|---|---|
| .NET SDK | **present, task-local** | `10.0.402` extracted to `C:\Users\choicky\worktmp\fusion-windows\settings-winui3-env\dotnet`; `dotnet --info` reports SDK 10.0.402, MSBuild 18.9.15 |
| SDK provenance | **verified** | `dotnet-sdk-10.0.402-win-x64.zip`, 300 755 826 bytes, SHA-512 `ae4fd528cb7e9f6af6aaf7de36715a69f761f183744f1bb3025d19f5a92447090ec561410e46266c1d10a63781ed6f3de771760bc76c876d87daabc367b898cd`, matching the official sidecar `.sha512` published beside the artifact |
| SDK source | `dotnetcli.blob.core.windows.net` | the only .NET download host that completes TLS from here; `builds.dotnet.microsoft.com` and `dot.net` redirect to it and fail |
| NuGet feed | **unreachable** | `dotnet restore` → `NU1301: 无法加载源 https://api.nuget.org/v3/index.json` → `The SSL connection could not be established` → `Received an unexpected EOF or 0 bytes from the transport stream` |
| Package cache | empty | no `~\.nuget\packages` |
| Windows App SDK | **cannot be restored** | `Microsoft.WindowsAppSDK` and `Microsoft.Windows.SDK.BuildTools` both resolve through `api.nuget.org` |
| Offline build | **works** | a zero-package `net10.0` project builds and runs with no feed configured; the SDK's bundled reference packs are enough |

The environment proxy is `http://127.0.0.1:9516`. The `CONNECT` tunnel to `api.nuget.org:443` is accepted
(`HTTP/1.1 200 Connection Established`) and the TLS handshake then fails, so the failure is at or after the tunnel, not at
DNS or at the socket. Its cause was not investigated further: no proxy setting, TLS validation, certificate store or
firewall rule was changed, and no credential was read.

**What this does and does not establish.** It establishes that a WinUI 3 assembly cannot be produced here, because
`Microsoft.WindowsAppSDK` is the only distribution of the WinUI 3 references and build tooling. It does **not** establish a
policy block: an `HTTP 000` and a `curl` exit 35 are a failed request, and several unrelated hosts (`github.com`,
`www.nuget.org`, `azuresearch-usnc.nuget.org`, `dotnetcli.blob.core.windows.net`) answer normally from the same shell.

## Consequence for this batch

- The **client core is implemented, compiled and tested** (`src/settings-winui3/FusionSettings.Core`, 215 checks). It has
  no package references, which is why it builds here.
- The **WinUI 3 presentation is not implemented**: it cannot be compiled, so writing it would produce source recorded only
  as COMPILE UNVERIFIED and still no deliverable. That is a deliberate boundary, not an omission.
- No package was produced, because a settings package without a settings executable would misrepresent the product. The
  installed `0.0.0-wp7+d095a44` package remains the shipped artifact.

## What would unblock the presentation

One environment change, on the owner's side: make a NuGet feed reachable that serves `Microsoft.WindowsAppSDK` and
`Microsoft.Windows.SDK.BuildTools` (a restored `api.nuget.org`, a mirror the owner approves, or a warm package cache in a
task-local directory). The SDK is already in place, so nothing else is missing.

Then, in order: the WinUI project → compile → bind the views to the tested core → package with the chosen deployment model
→ tray and Start Menu entries → owner acceptance.

## Not done to work around it

No package was fabricated, no binary relabelled, no dependency vendored from an unverified source, no mirror introduced,
and no check recorded as passed that did not run.
