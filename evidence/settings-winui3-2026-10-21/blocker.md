# Blocker — the WinUI 3 client cannot be built on this machine (2026-10-21)

Recorded before any client code was written, so that nothing downstream claims a build that did not happen.

## What is missing

| Requirement | State on this machine | Evidence |
|---|---|---|
| .NET SDK (any version) | **absent** | `dotnet --info` → `No SDKs were found`; `C:\Program Files\dotnet\sdk` does not exist |
| .NET runtimes | present (6.0.8, 8.0.15), runtimes only | `dotnet --list-runtimes` |
| SDK bundled with Visual Studio | absent (runtimes only) | `…\Visual Studio\18\Community\dotnet\` contains `net8.0/runtime`, `net10.0/runtime` |
| Windows App SDK (WinUI 3) package | not in the NuGet cache | `~\.nuget\packages` has no `microsoft.windowsappsdk` / `winui` |
| NuGet restore endpoint | **unreachable** | `api.nuget.org` → HTTP 000; via the configured proxy `http://127.0.0.1:2331` → curl exit 35 (SSL connect error) |
| NuGet CDN | **unreachable** | `globalcdn.nuget.org` → HTTP 000 |
| .NET SDK download hosts | **unreachable** | `dotnetcli.azureedge.net` → 000, `builds.dotnet.microsoft.com` → 000, `www.microsoft.com` → 000 |
| General network | partially up | `github.com` → 200, `dot.net` → 301, so this is host-level blocking, not a dead link |

WinUI 3 additionally needs the **Windows App SDK** package (and its runtime, framework-dependent or self-contained);
neither the package nor its restore feed is available, and the deployment model cannot be chosen without resolving it.

## What this blocks

- **WP3** — the client cannot be compiled, so there is no `FusionSettings.exe` replacement binary. Writing C# that cannot
  be compiled would be recorded as `COMPILE UNVERIFIED` and would still not produce the deliverable.
- **WP4** — tray/Start-Menu entry points would point at a non-existent executable; implementing them now would ship a
  broken launch path.
- **WP5** — no owner-test package can be produced, and no manifest/hash verification of a client that does not exist.

## What is not blocked

- **WP1** — the decision reconciling D085 with the owner's authorization (D087) and the bounded change contract.
- **WP2** — the migration inventory, read from the pinned schemas (`migration-inventory.md`).
- **WP6** — documentation reconciliation for what is actually true today.

## What would unblock it

Either of these, decided by the owner:

1. **Restore access** to `api.nuget.org` (and a .NET SDK download host), then install .NET SDK 8.0.x and restore
   `Microsoft.WindowsAppSDK` + `Microsoft.Windows.SDK.BuildTools` into the local cache.
2. **Pre-seed offline**: provide a .NET SDK installer plus a warm NuGet cache containing the Windows App SDK packages
   and their transitive dependencies.

Once either is true, the order is: client project → compile → the unit-level checks (draft model, key-binding parse and
serialisation, protocol client) → isolated Host round trip → packaging with the chosen deployment model → tray and
Start-Menu entry points.

## Not done to work around it

No package was fabricated, no pre-existing binary was relabelled, no dependency was vendored from an unverified source,
and no test was recorded as passed. The repository contains no client source in this batch.
