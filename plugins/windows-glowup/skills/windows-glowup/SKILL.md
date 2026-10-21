---
name: windows-glowup
description: Full Windows 10/11 PC tune-up - find what fills the disk, free RAM, trim startup apps and third-party services, clean dev build artifacts and caches, repair disk/system files, update drivers and apps. Use when the user says their disk or RAM is full, the laptop is slow, or asks to clean, speed up, debloat, refresh or "make Windows faster/cleaner".
---

# Windows Glow-Up

A safe, fast and complete Windows tune-up. It has three phases: **survey in parallel**, **agree a plan with the user**, then **act through scripts the user runs as Administrator**. The goal is that every gigabyte or megabyte of RAM freed is real, measured, and that nothing important breaks.

## Golden rules (learned the hard way)

1. **Measure, don't guess.** Report sizes from real commands. **Watch the locale:** in Indonesian or European locales `{0:N0}` prints `71.906 MB`, which means 71,906 MB (= 70 GB), not 71 MB. Use the bundled `survey.ps1`, which formats numbers invariantly in GB.
2. **Changing system settings is not something you do yourself.** Services, HKLM startup entries, chkdsk, DISM, uninstall: put them in a `.ps1` script, show it, and have the user run it from **PowerShell → Run as Administrator**. The Claude panel terminal is **not** admin. Every script must have an admin check at the top and print `GAGAL`/`FAILED` when a step fails (never a fake "OK").
3. **If the auto-mode classifier rejects a bulk delete** (e.g. `%TEMP%`), don't work around it. Give the user the command to run themselves.
4. **Before deleting, look at the target:**
   - List Docker containers and volumes before any purge (they may belong to other projects).
   - Check `git status` after cleaning node_modules/target to confirm no tracked files were lost.
   - Delete folders with `cmd /c rmdir /s /q`, which does not follow junctions (pnpm/workspace/worktree links). Never `Remove-Item -Recurse` on a folder that may contain a junction to shared data.
5. **Manual, not Disabled.** Third-party services go to `Manual` (they still start when needed). Startup entries are turned off through `...\Explorer\StartupApproved` (byte `03`), the same mechanism as Task Manager, so they can be switched back on. Always provide `undo.ps1`.
6. **Never touch:** `Windows\Installer`, `$PatchCache$`, WinSxS (only DISM `/StartComponentCleanup`), DriverStore, `pagefile.sys`, Recovery, `ProgramData\Package Cache`, WindowsApps (uninstall only), VC++ Redistributables (every year and both x86/x64 are needed), VS **Build Tools** + Windows Kits + `.rustup` when the user compiles Rust/C++, antivirus/Defender, audio/OEM drivers (Realtek, Nahimic on Legion, Lenovo Fn keys/Vantage), WSL/Hyper-V/HNS when Docker is used, anti-cheat (Riot Vanguard) when the user games, Print Spooler when a printer is present.
7. **A Windows service that is already Manual does not need to be disabled.** Windows starts and stops it itself. The real RAM savings come from third-party services, startup apps, and big apps the user leaves open (Electron apps, WizTree, browsers).

## Phase 1: Survey (fast, parallel)

Spawn **4 Explore agents at once** (one message, four `Agent` calls, `subagent_type: "Explore"`, medium breadth). Each one is read-only and returns a ranked table with real numbers. Tell each to use `scripts/survey.ps1 -Section <name>` where possible:

| Agent | Section | Look for |
|---|---|---|
| A, Disk | `disk` | Size of `%LOCALAPPDATA%\Temp` (especially hundreds of `edge-*` folders of ~0.5 GB each = leftover browser-automation profiles), npm/pnpm/bun/cargo/pip caches, Docker `docker_data.vhdx`, WSL `ext4.vhdx`, Claude `vm_bundles`, `C:\AMD`, `ProgramData\NVIDIA Corporation\Downloader`, `SoftwareDistribution\Download`, Steam workshop, top folders under Users/Program Files/ProgramData |
| B, Dev projects | `dev` | `scripts/find-regen.ps1 -Root <code folder>`: node_modules, .next, target, dist, .turbo, .venv, also inside `.claude\worktrees\*` (often 10× node_modules) |
| C, RAM & startup | `ram` | Top processes by RAM, Run/RunOnce/Startup folder entries + their StartupApproved state, running third-party services (not in `\Windows\`), duplicate tools (e.g. Lightshot + Snipaste + Snipping Tool) |
| D, Health | `health` | `Get-PhysicalDisk` health, TRIM, NTFS event 98 ("Spot Fix"/"Full Chkdsk"), recurring 7000/7009 errors (a service failing at every boot = 45 s slower boot), BSOD 41/6008, Windows Update failures, GPU driver date, BIOS version, `winget upgrade` list, duplicate products (e.g. 9 SQL Server instances, two VS editions) |

Combine the results into one ranked report. Present it in **three categories**:
- ✅ **Safe, changes nothing** (caches, leftover installers, build artifacts)
- ⚠️ **Optional, the user decides** (unused apps, WSL distro, Docker data, cache VMs, games)
- ⛔ **Do not delete** (with a one-line reason)

Ask the user to choose. Don't delete anything before they choose.

## Phase 2: Act

Do it yourself (no admin needed, user-level):
- Dev artifacts: `scripts/remove-regen.ps1 -Root <folder>` (node_modules/.next/target/dist; `.venv` is kept and reported). Afterwards check `git status` in each repo.
- User caches: `.bun\install\cache`, `npm-cache`, `pnpm`, `.cargo\registry`, `.cache\*` the user agreed to.
- Tell the user what needs reinstalling afterwards (`npm install`, `cargo build`, venv tooling under `target/`).

Generate a script for the user to run as Administrator (save it to `%USERPROFILE%\Documents\windows-glowup\`):
- `optimize.ps1` + `undo.ps1`: third-party services → Manual, startup → off. Services some vendors lock (ChatGPT Codex, Claude Cowork) fail even for admin. Report that honestly and leave them.
- `cleanup-cache.ps1`: Temp older than 2 days (skip the `claude` folder), `C:\AMD`, NVIDIA Downloader, SoftwareDistribution\Download (stop wuauserv/bits first), CrashDumps, JetBrains caches, browser caches (Chrome/Edge/Arc: `Cache`, `Code Cache`, `GPUCache`, `Service Worker\CacheStorage|ScriptCache`, `ShaderCache`, `GrShaderCache`, `component_crx_cache`). Skip a browser that is running.
- Duplicate or unwanted uninstalls: use the official uninstaller (`UninstallString` + `/S` or `/quiet`, `msiexec /x {GUID} /qn` in 2–3 passes for multi-component products like SQL Server, `vs_installer.exe uninstall --installPath ... --passive`). Then delete leftover folders and orphan entries in the Uninstall registry.
- `maintenance.ps1`:
  1. `Repair-Volume C -Scan`
  2. `Dism /Online /Cleanup-Image /RestoreHealth`
  3. `sfc /scannow`
  4. Set a service that fails at boot to Manual
  5. `Update-MpSignature`
  6. `winget upgrade --id <id> --exact --silent` for a **curated** list: security fixes (WinRAR, Git, VLC, Java, Zoom), runtime patches, dev tools, utilities. **Don't** auto-upgrade database majors (MongoDB/MySQL), language majors (Go, Anaconda), licensed IDEs, or game launchers.
- Docker: purge only after listing containers and the user agrees. Docker Desktop → Troubleshoot (bug/help icon) → **Clean up data** → tick **WSL 2** only. Use computer use (request access for `Docker Desktop` **and** `docker desktop.exe`).

## Phase 3: Repair the C: disk if NTFS 98 shows "Spot Fix"

On recent builds, `chkdsk C: /spotfix`, `Repair-Volume -SpotFix/-OfflineScanAndFix` and `fsutil dirty set` can all fail or not run at boot. The approach that has been proven to work:
1. In an admin cmd: `shutdown /r /o /t 0` (or Shift + Restart).
2. Troubleshoot → Advanced options → Command Prompt (the prompt starts with `X:\`; check the drive with `dir C:\Users`).
3. `chkdsk C: /f` → `exit` → Continue.
4. Verify: NTFS event 98 after boot = "Volume C: is healthy". WinRE doesn't write a Wininit report; that's normal.

## Things for the user to do themselves (give short steps)

- GPU driver: **NVIDIA App** from nvidia.com. Make sure the file is **not** `arm64` on an x64 CPU. Drivers → Download → Custom → Clean installation when the version jump is large.
- OEM: Lenovo Vantage / MyASUS / Dell Update for BIOS and drivers (with the charger plugged in).
- Laptop thermal mode: Balance + AI Engine as the default; Performance for heavy builds or gaming on AC; Quiet on battery.
- Electron apps (Logi Options+, Teams, etc.): close the window, the agent keeps running; or uninstall when the custom feature isn't needed.

## winget exit code cheat sheet

| Code | Meaning |
|---|---|
| `0` | OK |
| `0x8A15002B` (-1978335189) | No update applies; often already updated by another package |
| `0x8A150006` (-1978335226) | Installer failed; usually the app is running. Update from inside the app |
| others | Report as failed; don't call it a success |

## Final report

Close with a before/after table measured live: C: free, RAM free at idle, number of startup entries and services changed, disk health (event 98), updated apps (OK/failed). List what is still pending, and what the user has to reinstall or start manually (e.g. `net start postgresql-x64-18`, open Docker before using it).
