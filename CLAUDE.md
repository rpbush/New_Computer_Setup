# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A set of WinGet Configuration (DSC) YAML files plus a PowerShell bootstrapper that provisions a fresh Windows 11 machine: creates a Dev Drive, installs apps, applies Windows settings, and configures Office/Outlook. Target environment is Windows 11 + PowerShell; nothing here runs on Linux/macOS.

## Running

There is no build/test/lint pipeline — this is configuration code executed by `winget configuration`. To exercise it end-to-end:

1. Open Windows PowerShell, set execution policy (`Set-ExecutionPolicy -Scope Process Bypass`).
2. Copy `boot.ps1` to the user folder and run `.\boot.ps1`.
3. Reset execution policy after.

To run a single DSC file in isolation (useful when iterating on one workload):

```powershell
winget configuration -f .\rpbush.dev.dsc.yml
```

## Architecture

### Two-phase elevation in `boot.ps1`

`boot.ps1` runs twice — first as the regular user, then re-launches itself elevated via `Start-Process PowerShell -Verb RunAs`. The `if (!$isAdmin)` branch runs the non-admin workload (`generalSoftware`), and the `else` branch runs the admin-only workloads (`office`, `dev`, PowerToys). This split exists because some WinGet packages refuse to install elevated (e.g. Spotify) while others require admin. When editing, keep packages on the correct side of the elevation boundary.

`GetLatestWinGet` runs before either branch — it parses `winget -v`, and if the version is < 1.6, downloads and sideloads the VCLibs / UI.Xaml / DesktopAppInstaller `.appx` bundles. This is the only reason the script handles a "no winget yet" state; everything downstream assumes a working `winget`.

### DSC files are fetched from a remote URL, not local

`boot.ps1` does **not** consume the YAML files sitting next to it on disk. It constructs URLs against `https://raw.githubusercontent.com/rpbush/New_Computer_Setup/main/` and passes those to `winget configuration -f <url>`. This means edits to local YAML will not take effect until pushed to `main`.

The PowerToys DSC is the one exception: it is referenced as a local path on `Z:\source\powertoys\.configurations\configuration.vsEnterprise.dsc.yaml`, which only exists after `rpbush.dev.dsc.yml` has cloned PowerToys to the Dev Drive.

### The four DSC workloads

- `rpbush.generalSoftware.dsc.yml` — non-admin: PowerToys, 1Password, Logitech Options+, Spotify.
- `rpbush.office.dsc.yml` — admin: Office, Teams, plus a `PSDscResources/Script` block that writes `HKCU\Software\Microsoft\Office\16.0\Outlook` registry keys to configure auto-login, then starts Outlook/Teams to hydrate them. Order matters: Office/Teams install resources have `dependsOn` from the script resources.
- `rpbush.dev.dsc.yml` — admin: shrinks Disk 0 and formats a 75 GB ReFS **Dev Drive on `Z:`** (label "Dev Drive", `AllowDestructive: true`), installs Git, clones PowerToys to `Z:/Source/`, installs VS Code and PowerShell 7. The Dev Drive letter `Z` is referenced by the PowerToys local path in `boot.ps1` — changing it requires updating both.
- `rpbush.winSettings.dsc.yml` — File Explorer / Taskbar tweaks, plus PSDscResources scripts that iterate Store packages to force updates and trigger Windows Update via COM (`Microsoft.Update.Session`). Runs last in the admin branch so any pending reboot lands after package installs.

### readme.md TODO list

`readme.md` contains a long list of manual Windows settings (taskbar pins, notification toggles, audio device renames, etc.) that have **not yet been automated**. Treat that section as a backlog of candidate DSC additions, not as documentation of existing behavior.
