# Build prerequisite — the WinUI 3 client cannot be compiled on this machine (2026-10-21)

Recorded so that nothing downstream claims a build that did not happen. It is a **development-environment prerequisite**,
not a product decision and not a statement about the design.

## What was observed

| Requirement | State | Observation |
|---|---|---|
| .NET SDK (any version) | absent | `dotnet --info` → `No SDKs were found`; no `C:\Program Files\dotnet\sdk` |
| .NET runtimes | present (6.0.8, 8.0.15) | runtimes only; runtimes cannot compile |
| .NET SDK component in Visual Studio 18 Community | absent | `vswhere -requires Microsoft.NetCore.Component.SDK` returns nothing; the VS `dotnet\` folder holds `net8.0/runtime` and `net10.0/runtime` only |
| NuGet configuration | none | no user `NuGet.Config`; no `~\.nuget\packages` cache at all |
| Windows App SDK (WinUI 3) package | absent | nothing in the package cache |
| Restore/download requests | **fail** | `api.nuget.org`, `globalcdn.nuget.org`, `dot.net/v1/dotnet-install.ps1`, `aka.ms/dotnet/8.0/...` all return HTTP 000 with `curl` exit 35 (SSL connect error); the environment sets `http_proxy`/`https_proxy` to a local address |
| Other hosts | reachable | `github.com` returns 200, `www.nuget.org` returns 200 (the site, not the API) |

**What this does and does not establish.** It establishes that *this* environment currently cannot obtain a .NET SDK or
the Windows App SDK packages, therefore cannot compile a WinUI 3 client. It does **not** establish that the hosts are
blocked by policy: a failed TLS connection through a local proxy is a failed request, and its cause (proxy rule, TLS
interception, DNS, certificate store) was not investigated. No proxy setting, certificate validation or firewall rule was
changed, and no credential was read.

## Consequence for this batch

- The client is **not implemented, compiled, packaged or installed**. No C# was written, because unbuildable source would
  still not produce the deliverable and would be recorded only as COMPILE UNVERIFIED.
- The tray entry-point change does **not** depend on it and was implemented and tested separately (fusion-windows PR #30).

## What would unblock it

Either, as an environment change the owner controls:

1. Install a .NET SDK (any supported version; 8.0.x is a reasonable default but not a requirement of the design) and
   restore `Microsoft.WindowsAppSDK` plus `Microsoft.Windows.SDK.BuildTools` from a reachable feed; or
2. Provide an SDK installer and a warm NuGet cache in the approved task-local location.

Then: client project → compile → unit checks (draft model, key-binding parse/serialise, protocol client) → isolated Host
round trip → packaging with the chosen deployment model → entry points. The tray's settings launch path already works with
the current executable, so nothing in the shipped product waits on this.

## Not done to work around it

No package was fabricated, no binary relabelled, no dependency vendored from an unverified source, and no test recorded as
passed.
