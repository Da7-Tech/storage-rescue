# Storage Rescue on Windows

This reference replaces the macOS workflow when cleaning Windows storage. Use native PowerShell. Keep the scope to storage the user has placed in scope. Installing the skill does not authorize cleanup.

## Authorization and protected data

Survey read-only first, then present exact items, sizes, ownership, proposed action, and recovery cost. Obtain explicit approval per cleanup category and per item for personal data. Existing specific approval persists; a vague request such as "clean all junk" is not item approval. Use a question tool if available and appropriate; otherwise ask in chat and wait. Silence is not approval. Never preselect deletion of user files. Ask separately before emptying the Recycle Bin, formatting a drive, uninstalling software, stopping an app, or permanent deletion. Never format as a prerequisite to offloading without informed authorization.

Protect credentials, .env files, SSH/GPG keys, password managers, browser profiles, mail, photos, chat/session databases, active worktrees, and app settings. Never manually remove Windows, WinSxS, Windows Installer, Program Files, ProgramData, Recovery, System Volume Information, WindowsApps, pagefile.sys, hiberfil.sys, restore points, or a .git directory. Do not delete agent configuration, runtimes, skills, MCP installations or uvx environments merely because they look like caches. AppData contains user data as well as caches; it is never a whole-folder cleanup target.

Use owner-supported cleanup before path deletion. Close apps before clearing their caches; identify processes with Get-Process or Get-CimInstance Win32_Process, exposing only relevant metadata and never credential-bearing command lines. Check PATH, PowerShell profiles, scheduled tasks, services, app configuration and running tools for dependencies before removing runtimes or projects. Do not kill unrelated processes or auto-elevate. If administrator rights are required, explain the operation and let the user perform the elevated step.

## Intake and survey

Determine whether the user wants offload to an external drive or local reclaim, which categories are in scope, and which active projects/apps must remain. Record answers and avoid reopening settled choices without new evidence. Apps, Docker, WSL and VM data are excluded until explicitly included.

Measure actual volume free space before and after using:

```powershell
Get-Volume | Select-Object DriveLetter, FileSystemLabel, FileSystem, Size, SizeRemaining, HealthStatus
Get-CimInstance Win32_LogicalDisk -Filter 'DriveType=3' |
  Select-Object DeviceID, Size, FreeSpace
Get-ChildItem -LiteralPath $env:USERPROFILE -Force |
  Select-Object Name, FullName, Attributes, LastWriteTime
```

Resolve the user's real Documents/Desktop/Downloads locations, including redirected folders; do not assume they live under the profile. Survey selected local directories progressively. Do not recursively scan every drive or all AppData by default. Enumerate iteratively with Get-ChildItem -LiteralPath on each directory. Check attributes BEFORE opening or descending: skip ReparsePoint entries (junctions, symlinks, cloud placeholders) and record them; also skip Offline/RecallOnDataAccess/RecallOnOpen entries where present. Do not use -FollowSymlink. Do not hash OneDrive or other cloud placeholders during a survey because opening them can download content. Ask before hydrating or moving cloud-managed data.

Sum ordinary file Length values for estimates, record access-denied errors and unreadable subtrees, and label incomplete totals. Do not claim LastWriteTime is last-used time; LastAccessTime may be disabled or unreliable. Logical lengths are not allocated bytes: hard links, NTFS compression, sparse files, deduplication and cloud placeholders distort totals. Report measured change in volume free space as reclaimed space; folder-size estimates are not proof.

Look for large media, archives, installers, build output, old node_modules, local models (.gguf/.safetensors), package caches, backups, agent chat histories and VM disks. Classify findings: A regenerable cache; B costly/re-downloadable; C user data; D protected/in use. Unknown items remain untouched pending identification.

## Reclaim catalog

Use Windows Settings > System > Storage > Temporary files or Cleanup recommendations for Windows-managed cleanup. Review the selected categories with the user, especially Downloads, Recycle Bin and Previous Windows installations. Removing Windows.old removes rollback capability. Storage Sense policies affect future cleanup and need separate approval. Do not manually clean WinSxS or use DISM /ResetBase. Avoid introducing scheduled cleanup unless requested.

For package caches, first locate and measure the actual configured cache/store. Examples: npm cache verify is diagnostic; pnpm store prune removes unreferenced store content; pip cache purge and uv cache clean clear their respective caches. Confirm tool availability and the installed version's exact cleanup scope before execution. Do not assume npm cache clean --force removes download data only: it can remove the entire configured cache, including _npx executable environments. Distinguish the npm download cache from _npx environments and approve them separately. Before clearing _npx, check running processes and configuration references to its entries; skip active environments. Use bounded, scoped cleanup rather than a blanket cache wipe. uv cache cleaning can remove live uvx tool environments and MCP runtimes. Do not clean it while those are in use. NuGet caches can be cleared with dotnet nuget locals all --clear after checking active builds. Explain that tools may need downloads afterward.

For browser/editor caches use the application's Clear cache UI, after closing relevant apps when required. Preserve sign-ins, cookies, settings, extension data and local databases unless specifically included. Use uninstallers for old apps and SDKs; do not remove their installation folders by hand.

For temp files, resolve [IO.Path]::GetTempPath(), list exact candidates, check ownership and active use, then delete only the approved entries. Skip locked/denied files and report them. Never recursively delete the temp root or use a wildcard against a broad directory. Do not take ownership, change ACLs, or kill processes to force deletion.

For Docker, inspect docker system df first. List the concrete images/build cache/container candidates before pruning. Volumes and stopped containers can contain user data. Do not run docker system prune --volumes or broad destructive prune commands from a generic cleanup request. Use the installed tool's supported, scoped operation after approval.

For WSL/Hyper-V/Docker virtual disks: inspect wsl --list --verbose and owner settings. Never delete ext4.vhdx, Docker VHDX, or VM disks manually. Deleting files inside a guest may not shrink its host VHDX. Compaction is a separate operation needing an owner-supported procedure for the installed version, a backup, and shutdown of the affected workload with approval. Never unregister a distribution to reclaim space.

For OneDrive, prefer its Free up space action when the user wants an online-only copy, after verifying synchronization and explaining offline availability. Deleting a synced file can delete it in the cloud and on other devices. Do not treat cloud files as disposable local caches.

## Recycle Bin and exact-path operations

Personal files approved for removal go through File Explorer's Recycle Bin operation. An API may be used only if it supports Recycle Bin semantics with visible error handling; Microsoft.VisualBasic.FileIO.FileSystem.DeleteFile/DeleteDirectory with SendToRecycleBin is one possible supported route. Never silently fall back to permanent deletion if recycling fails or is unavailable (network/removable volumes and oversized items may behave differently). A staging folder on the same volume is recoverable but does not free that volume's space. Explain this if used as a temporary holding area.

Before any recursive move/delete, resolve each exact absolute path and verify it is a strict descendant of the explicitly approved target directory, on the expected volume, and not a reparse point. Recheck immediately before mutation. Do not target a drive root, profile root, whole active workspace or unresolved variable. If an approved item is an inactive project or workspace, resolve that exact item and apply the same dependency, inventory and authorization checks; a workspace name alone is not evidence that it is disposable. Use -LiteralPath. Do not enumerate in PowerShell then send paths to cmd/batch for deletion. Stop on unexpected links, new files, access errors or identity changes. Clear-RecycleBin permanently removes data and requires separate explicit approval for the affected drive/content; never run it as a routine closing step.

## Offload protocol

1. Identify the external device by disk/volume identity, label, filesystem and capacity, not drive letter alone. Check it is healthy, writable, distinct from the source volume and has a space margin. Do not format it or erase existing data. NTFS can preserve Windows metadata better than exFAT; exFAT lacks NTFS ACLs, alternate data streams, EFS and some link semantics. FAT32 has a 4 GiB file limit. Inventory ACLs, streams, encryption, links, sparse/compressed files and long paths before choosing a method. Never claim ZIP or exFAT preserves all metadata. Use a verified metadata-preserving archive/backup or retain the source when fidelity cannot be established. EFS needs a tested recovery certificate/key backup before proceeding.
2. Create a concrete plan and manifest outside the source tree: approved source paths and inventories, volume identities, destinations, byte lengths, SHA-256 hashes, timestamps, metadata limitations, approval scope and per-file status. Keep existing destination files untouched; use a unique destination directory. Include hidden files and project .git data when the whole project is approved. Detect unreadable files and links rather than silently excluding them.
3. Copy without deleting, using Copy-Item for suitable ordinary files or robocopy with deliberately chosen options. Never use /MIR, /PURGE, /MOVE or /MOV. /J requests unbuffered file copying; it does not make subsequent hashing unbuffered. Set bounded retry counts such as /R:2 /W:2. /XJ excludes junctions but does not solve every symbolic-link/metadata case. Check robocopy's bitmask exit status: values below 8 are not necessarily an exact match; inspect mismatch/extra-file flags and independently verify the manifest. Keep long copies in a hidden background process with a log, and wait for confirmed completion before verification or disconnection.
4. Hash every destination with Get-FileHash -Algorithm SHA256 and compare to the recorded source hashes. A matching size or successful copy is insufficient. A normal Get-FileHash read may use the Windows cache: do not call it a cache-bypassed read. If verified uncached readback and flushing are unavailable, report that limitation and retain sources until a physical reconnect and full second readback succeeds.
5. After all copy handles are closed, ask the user to safely eject the external drive through Windows, then physically reconnect it. Never forcibly dismount a system/in-use volume or use mountvol/diskpart as an automatic substitute. Reidentify the drive by device/volume identity; its letter may change. Re-hash every destination after reconnect. If reconnect or any hash verification fails, keep every affected source and report the reason.
6. Before removing any approved source, re-enumerate it and compare the complete inventory, metadata and source SHA-256 hashes with the manifest. New/changed/unreadable entries remain. Verify destination identity again. Only entries with both destination verification passes and unchanged sources are eligible. Preserve user authorization requirements; permanent deletion needs explicit approval, otherwise recycle. Empty the Recycle Bin only with separate approval; until then do not promise its occupied bytes have been reclaimed. Keep the manifest and restore instructions on the drive and a local copy. Do not remove whole directories containing unverified entries.
7. Measure volume free space again, report copied/verified/recycled/permanently removed amounts separately, and explain when the external drive holds the only copy. Recommend another backup. Restore by manifest, verify the copied-back hashes and metadata, and test the owning application.

## Completion

Report measured free space before/after, estimates with uncertainty, approved actions taken, skipped/in-use/unknown items, manifest locations, restore steps and any expected cache rebuilds. Validate affected apps/tools without launching unrelated services or sending user data externally. Do not infer that reaching a requested free-space target authorizes deleting protected or unapproved data.
