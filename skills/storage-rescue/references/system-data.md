# "System Data" explained

In System Settings > General > Storage, "System Data" is everything macOS does not put in a named category. It is not one folder and it is not only temporary files. On a developer's Mac it is often more than 150 GB. The figure is also recalculated slowly, so it can lag behind cleanups by hours or until a restart.

## Measure the parts

```bash
df -h /System/Volumes/Data
diskutil info / | grep -i "Container Free Space"   # differs from df by purgeable space
tmutil listlocalsnapshots /
sysctl vm.swapusage
ls -la /private/var/vm
du -xsh ~/Library ~/Library/"Application Support" ~/Library/Containers ~/Library/"Group Containers" ~/Library/Developer 2>/dev/null
du -xsh ~/.[!.]* 2>/dev/null | sort -rh | head -20       # hidden folders in the home
du -xsh /private/var/folders /private/var/db /Library 2>/dev/null
diskutil apfs list | grep -E "Name:|Capacity Consumed"    # simulator runtimes appear as separate volumes
xcrun simctl runtime list 2>/dev/null
```

Some of these need Full Disk Access for complete numbers; report a figure as partial if access was denied.

## Typical parts and what to do

| Part | Where | Can it shrink safely? |
|---|---|---|
| App data | `~/Library/Application Support`, `Containers`, `Group Containers` | Often the largest share. Treat per app (`app-data.md`) |
| Hidden tool folders | `~/.cache`, `~/.local`, `~/.npm`, `~/.gradle`, `~/.android`, `~/.colima`, `~/.docker`, agent folders | Caches yes (`reclaim-catalog.md`); environments and data need checks |
| Simulator runtimes and devices | `xcrun simctl runtime list`, `~/Library/Developer/CoreSimulator` | Only with approval after listing the devices and their data, including devices marked unavailable |
| Per-user temp and caches | `/private/var/folders/...` (`$TMPDIR`, `C/`, `X/`) | A restart is the safe way. Exception: stale `code_sign_clone` folders of a fully quit app |
| Swap and sleep image | `/private/var/vm`, the VM volume | Managed by macOS. Shrinks after closing heavy apps or restarting. Never delete |
| Local Time Machine snapshots | `tmutil listlocalsnapshots /` | macOS frees them under pressure. Forced deletion is a user decision and needs `sudo` |
| Prepared OS update | Snapshots named `com.apple.os.update-*` | Installing the pending update releases the space |
| Logs and diagnostics | `/private/var/db/diagnostics`, `~/Library/Logs` | Old user logs yes; system logs are rotated by macOS |

## How to explain it to the user

Give a short table of the measured parts with sizes, say which ones were already reduced, which ones a restart or an update will reduce, and which ones are app data they chose to keep. Do not promise that the Storage panel will show the new figure immediately.
