# Reclaim catalog

What can be removed, how, and under which condition. Measure each item with `du -sh` before proposing it, and only propose items that exist on this Mac. Tools that are not installed are skipped silently. Commands that are not installed or fail are reported as skipped, not as done.

## Before removing anything that is not a pure cache

Search for references to the path:

```bash
P="$HOME/.some-folder"
N="$(basename "$P")"   # search by name: configs write the path as ~/..., $HOME/..., or absolute
grep -sIl -- "$N" ~/.zshrc ~/.zprofile ~/.zshenv ~/.bashrc ~/.bash_profile ~/.profile 2>/dev/null
grep -rsIl -- "$N" ~/.config ~/Library/LaunchAgents 2>/dev/null | head
lsof +D "$P" 2>/dev/null | head
```

Read each match to confirm it really refers to this folder. If anything references it, keep it or ask.

When checking for running processes with `pgrep -f`, the pattern can match the checking command itself. Use `ps -axo pid,args | grep '[G]radleDaemon'` (the bracket stops the match on itself) or confirm each PID with `ps -p <pid> -o args=`.

## Tier A: regenerable

Remove after category approval. The program rebuilds or re-downloads these on demand. Expect the next run of that tool to be slower.

| Item | Command or path | Precondition | Notes |
|---|---|---|---|
| Trash | Finder > Empty Trash, or `osascript -e 'tell application "Finder" to empty trash'` | Separate explicit approval | An agent without Full Disk Access may see `~/.Trash` as empty. Do not report it as empty if it could not be read |
| Homebrew | `brew cleanup -n -s --prune=all` (preview), then the same without `-n` | No `brew` command running | Removes old versions and cached downloads Homebrew considers outdated. Some downloads of installed packages can remain |
| npm | `npm cache clean --force` | No `npm install` running | |
| pnpm | `pnpm store prune` | No install running | Do not delete `PNPM_HOME`; it holds globally installed binaries and is on `PATH` |
| Yarn | `yarn cache clean` | | |
| Bun | `bun pm cache rm` | | |
| pip | `pip3 cache purge` | | |
| uv | `uv cache prune` (removes unused entries) or `uv cache clean` | No `uv` or `uvx` process running: `pgrep -fl 'uv|uvx'` | Environments of `uvx`-launched tools live in this cache. Removing them breaks running tools (for example MCP servers) until restarted. List them first and rebuild after |
| Go build cache | `go clean -cache` | | `go clean -modcache` is tier B: it re-downloads every module |
| Gradle | delete `~/.gradle/caches` | No Gradle daemon: `ps -axo pid,args \| grep '[G]radleDaemon'` prints nothing, or run `gradle --stop` | |
| CocoaPods | `pod cache clean --all` | | |
| Playwright browsers | `npx playwright uninstall --all` | No test run in progress | Re-install with `npx playwright install` |
| Puppeteer, electron-builder, node-gyp caches | `~/.cache/puppeteer`, `~/Library/Caches/electron-builder`, `~/Library/Caches/node-gyp` | | |
| Xcode DerivedData | `~/Library/Developer/Xcode/DerivedData` | Xcode closed | Rebuilt on next build |
| App updater leftovers | `~/Library/Caches/*.ShipIt`, `~/Library/Caches/*updater*` | The app is not updating right now | Downloaded update packages |
| Per-app caches | `~/Library/Caches/<bundle-id>` | That app closed | Prefer the app's own "Clear cache" when it has one. Leave `com.apple.*` folders to the system |
| Old app logs | Files in `~/Library/Logs` older than 14 days | | Keep logs the user or this skill is using (for example the offload manifest folder) |
| Old temp files | Regular files in `$TMPDIR` older than 7 days, excluding `*.sock`, `*.lock`, `*.pid` | | A restart clears most of `$TMPDIR` more safely than manual deletion |
| Old CLI versions | For example `~/.local/share/claude/versions/*`, other agents' `versions/` folders | Identify the current version from the launcher symlink | Keep the current version and one previous |
| Stale code-sign clones | `$TMPDIR/../X/<bundle-id>.code_sign_clone/code_sign_clone.*` | That app fully quit (no process at all) | These are APFS clones of the app. `du` shows their full size, but deleting them frees much less. Measure with `df` |

Use `find ... -mtime +N` with `-type f` for age-based cleanup, and print the count and total size before deleting. If the host blocks a broad `rm -rf` or `find -delete`, delete the exact listed paths with a short script instead of trying to bypass the guard.

## Tier B: re-downloadable or costly

Explain the cost of getting it back. Remove only on explicit approval. In Offload mode, consider moving instead.

| Item | How to inspect | How to remove | Cost of getting it back |
|---|---|---|---|
| Simulator runtimes | `xcrun simctl runtime list`; devices per runtime: `xcrun simctl list devices -j` | Delete the devices on that runtime (`xcrun simctl delete <udid>`), then `xcrun simctl runtime delete <id>` | Gigabytes of download; device data on that runtime is lost. Confirm the device list with the user |
| Unavailable simulator devices | `xcrun simctl list devices unavailable` and each device's folder size under `~/Library/Developer/CoreSimulator/Devices/<udid>` | `xcrun simctl delete <udid>` per approved device, or `xcrun simctl delete unavailable` when the user approves all of them | "Unavailable" means not supported by the current Xcode SDK, not that the data is worthless. Installed apps and their data on those devices are lost |
| Docker or Colima | `docker system df` | `docker builder prune`, `docker image prune` (dangling only). `-a` or volume pruning only with item approval | Images re-pulled; volumes may hold databases (user data) |
| Claude Desktop VM | `~/Library/Application Support/Claude/vm_bundles` | Delete with Claude quit | Re-downloaded (around 10 GB) the next time its agent or work mode is used. Normal chats do not need it |
| Local AI models | Ollama: `ollama list`; Hugging Face: `hf cache ls` (older installs: `huggingface-cli scan-cache`); LM Studio: its model manager | `ollama rm <model>`; `hf cache rm <id>` (older: `huggingface-cli delete-cache`); LM Studio UI. Check `--help` of the installed version | Large downloads. Check which model an app is configured to use first; never remove the active one without saying so |
| Android emulators and system images | `~/.android/avd`, `~/Library/Android/sdk/system-images` | `avdmanager delete avd -n <name>`, Android Studio SDK Manager | Re-download; emulator data lost |
| Go modules, Cargo registry | `~/go/pkg/mod`, `~/.cargo/registry` | `go clean -modcache`; delete `~/.cargo/registry/cache` | Full re-download on next build |
| Time Machine local snapshots | `tmutil listlocalsnapshots /` | macOS deletes them when space is needed. To force it, the user runs `sudo tmutil deletelocalsnapshots <date>` | Loses local restore points |
| Downloaded but uninstalled macOS update | System Settings > General > Software Update | Install the update | Space returns after installation |

## Tier C: user data

In Reclaim mode, only on item-by-item approval, and to the Trash, keeping the relative path, unless the user explicitly asks for permanent deletion:

Run this as one script (for example `bash trash-items.sh`). It stops if the Trash folder cannot be created:

```bash
# one folder per operation, never reused; items go under items/, the log stays outside it
D="$(mktemp -d "$HOME/.Trash/storage-rescue-$(date +%Y%m%d-%H%M%S)-XXXX")" || { echo "cannot create a Trash folder"; exit 1; }
case "$D" in "$HOME/.Trash/storage-rescue-"?*) ;; *) echo "unexpected Trash folder: $D"; exit 1 ;; esac
mkdir "$D/items" || exit 1
for REL in "relative/path/one" "relative/path two"; do   # paths relative to $HOME
  case "/$REL/" in //*|*/../*|*/./*) echo "REFUSED: $REL"; continue ;; esac
  mkdir -p "$D/items/$(dirname "$REL")" || { echo "NOT MOVED: $REL"; continue; }
  # an earlier item may have created this path (for example "a/b" before "a"); mv would then nest inside it
  if [ -e "$D/items/$REL" ] || [ -L "$D/items/$REL" ]; then echo "NOT MOVED (overlaps an earlier item): $REL"; continue; fi
  mv -n -- "$HOME/$REL" "$D/items/$REL"
  if [ -e "$HOME/$REL" ] || [ -L "$HOME/$REL" ] || { [ ! -e "$D/items/$REL" ] && [ ! -L "$D/items/$REL" ]; }; then
    echo "NOT MOVED: $REL"
  else
    printf '%s\t%s\n' "$HOME/$REL" "$D/items/$REL" >> "$D/moved.tsv"
  fi
done
echo "Trash folder: $D"
```

`mv -n` never overwrites. The script refuses absolute paths and `.` or `..` components, and the check after each move catches a move that did not happen. `moved.tsv` records where each item came from; it sits outside `items/`, so no moved file can share its name. Finder's "Put Back" may not work for items moved this way; the TSV is the restore map.

Common candidates to point out, never to decide on alone:

- Installer files (`.dmg`, `.pkg`, `.xip`, `.apk`, `.ipa`) whose app is already installed. Check `/Applications` and `~/Applications` first.
- Byte-identical duplicates (same size and SHA-256). Keep the copy in the more meaningful location.
- Old exports, screen recordings, and large videos.
- Backups the user made of app data (`.bak`, `*-backup-*`, zipped snapshots) after confirming the live data is fine.
- Old project folders: suggest deleting only their `node_modules`, `venv`, or build output when a lockfile or requirements file exists, so the project can be rebuilt.

## Tier D: never touch

Photos library, Mail, Messages, iCloud Drive placeholders, Keychains, `~/.ssh`, `~/.gnupg`, `.env` and credential files, `.git` folders, password-manager data, `/System`, `/Applications/*.app` bundles, `/private/var/vm` (swap and sleep image), `/private/var/db`, anything a running process holds open.
