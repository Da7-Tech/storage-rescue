# App data: chats, media caches, VMs, models

Apps often hold the largest single items on a Mac: chat and session databases of editors and coding agents, messenger media caches, VM images, and local models. These are not plain caches. Handle them with the app's own tools first, and with a verified backup when the data matters.

## 1. Find out what the data is

- Locate the folder (`~/Library/Application Support/<App>`, `~/Library/Group Containers/<id>`, `~/Library/Containers/<id>`, `~/.<tool>`).
- Find the biggest files: `du -ah "<folder>" 2>/dev/null | sort -rh | head -20`.
- For SQLite databases, read the structure without changing anything: `sqlite3 -readonly <db> ".tables"`, then count rows or bytes per table or key prefix to see what holds the space.
- Check when the app was last used and whether it is running.

Explain to the user what the data is (for example "conversation history of an AI editor", "downloaded photos and videos from chats", "a Linux VM used by an agent mode") and what they lose by removing it.

## 2. Choose the least destructive route

1. **The app's own cleanup.** Messengers usually have a storage screen that clears downloaded media while keeping messages in the cloud. Example for Telegram on macOS: Settings > Data and Storage > Storage Usage > Clear Cache, repeated for each account. Media in secret chats exists only on the device.
2. **The app's own export or archive feature**, if the user wants to keep a history.
3. **Offload a full backup, then clear inside the app or the database.** Use this when the app has no cleanup screen and the user wants the history kept somewhere.
4. **Delete a re-downloadable component.** For example, an agent VM bundle that the app downloads again on demand.

If GUI automation of the app's cleanup screen is unreliable, give the user the exact clicks, then measure after they confirm.

## 3. Full backup before clearing a database

Use this procedure when the user approved backing up and then clearing an app's chat or session history.

1. Quit the app completely. Confirm with `pgrep -x "<App>"` and `pgrep -fl "<App>"`.
2. Copy the whole data folder that contains the database, including `-wal` and `-shm` files, to the drive (or another location the user chose), using the verified copy in `offload-protocol.md`. Write a `checksums.json` next to the backup.
3. Open the backup copy read-only and confirm it is valid: `sqlite3 -readonly <backup-db> "PRAGMA integrity_check;"` returns `ok`, and the expected tables have rows.
4. Clear only the history, not settings or sign-in state:
   - Identify the tables or key prefixes that hold conversations (for example message, bubble, composer, or session keys) and the ones that hold settings and authentication.
   - Delete only the conversation rows, inside a transaction, then `VACUUM`.
   - Run `PRAGMA integrity_check;` again.
   - Move, do not delete, secondary history files such as search indexes, to a temporary folder until the app has been tested.
5. Open the app. Confirm it starts, the user is still signed in, and their preferences remain. Then quit it again if it was closed before.
6. Record in the drive README where the backup is and how to restore it: quit the app, copy the backup folder back over the data folder, open the app.

If the database schema is unknown or ambiguous, stop and ask. Clearing the wrong table can sign the user out or erase settings. Backing up and leaving the database as is remains a valid choice.

## 4. Re-downloadable components

| Component | Where | Notes |
|---|---|---|
| Agent or sandbox VM bundles (for example Claude Desktop `vm_bundles`) | The app's support folder | Re-downloaded on next use of that mode. Quit the app first |
| Old versions of an app's bundled CLI or agent | `versions/` or `_versions/` folders | Keep current and one previous |
| Electron `Cache`, `Code Cache`, `GPUCache`, `CachedData` | The app's support folder | App closed. Do not touch `Local Storage`, `IndexedDB`, or `Session Storage`; they hold sign-in and app state |

## 5. Local models used by an app

Read the app's configuration to find which model it uses. Keep the active or primary model on the Mac. When moving others, record in the README which app used each one and the exact original path, so it can be restored to the same place. After restoring a model, test the feature that uses it, not just the file's presence.
