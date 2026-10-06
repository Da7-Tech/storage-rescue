# External drives

## Identify the drive

```bash
diskutil list external physical
diskutil info /dev/diskNsM | grep -E "Volume Name|File System Personality|Type \(Bundle\)|Mount Point|Read-Only|Volume Free Space|Protocol"
mount | grep /Volumes/
```

Confirm with the user that the identified partition is the drive they mean (model, capacity, volume name) before writing to it.

## File systems

| File system | macOS can write? | Max file size | Notes |
|---|---|---|---|
| APFS | Yes | Practically unlimited | Best for Mac-only use. Keeps permissions, symlinks, extended attributes |
| HFS+ (Mac OS Extended) | Yes | Practically unlimited | Fine on spinning disks |
| exFAT | Yes | Practically unlimited | Works on Mac and Windows. No symlinks or Unix permissions; pack projects with `tgz` |
| FAT32 (MS-DOS) | Yes | 4 GiB minus 1 byte | Files of 4 GiB or more cannot be copied; pack and split, or use another drive |
| NTFS | Read only | Practically unlimited | Needs a helper to write; see below |

Never reformat a drive that holds data unless the user explicitly asks and the data is backed up elsewhere. Reformatting erases everything on it.

## Writing to an NTFS drive

macOS mounts NTFS read-only. Options, in order of preference when the drive already holds data:

1. **A free, open-source helper: `anylinuxfs`** (Apple Silicon Macs only; check that `uname -m` prints `arm64`, and on Intel Macs use option 2 or 3). It runs a small Linux virtual machine that mounts the drive with Linux's NTFS driver and shares it back to macOS over NFS. Install with Homebrew:

   ```bash
   brew tap nohajc/anylinuxfs
   brew trust nohajc/anylinuxfs
   brew install anylinuxfs
   ```

   Recent Homebrew versions refuse to load formulae from a third-party tap until the user trusts it. `brew trust` records that decision; it is the user's call, so explain it and ask before running it, and never disable Homebrew's trust check instead.

   Mounting needs raw disk access, so the user runs it in their own Terminal (macOS privacy controls usually block an agent's process from it):

   ```bash
   diskutil list external physical          # find the partition, for example disk9s1
   sudo anylinuxfs mount -r --ignore-permissions /dev/disk9s1
   ```

   `-r` allows mounting even though macOS already mounted the partition read-only. `--ignore-permissions` makes every file appear owned by the current user. The volume then appears under `/Volumes/<name>` as an `nfs` mount. Check with `mount | grep /Volumes/`.

   Known behavior: long-held file descriptors on this mount can go stale (`ESTALE`, errno 70). Re-open files per write and retry. Reads can also be served from the helper VM's own cache, which `F_NOCACHE` on the Mac does not bypass. That is why the offload protocol re-verifies after unmounting and mounting again, before deleting any source.

   To cycle the mount between the copy phase and the re-verification phase: `anylinuxfs unmount /Volumes/<name> --wait-for-vm`, confirm it is gone with `mount | grep /Volumes/<name>`, then the user mounts it again with the same `sudo anylinuxfs mount ...` command.

2. **A commercial NTFS driver** the user already owns. Follow its vendor's instructions.

3. **Reformat to exFAT or APFS**, only for an empty drive or after the data is safely elsewhere.

The disk number (`disk9`) can change every time the drive is connected. Always re-check with `diskutil list` before mounting.

## Test before the real run

1. **Write test.** Create a folder on the drive, write a small file, read it back, delete it.
2. **Speed test.** Write and read back a 1 to 2 GB test file with the cache disabled, to estimate how long the plan will take. Delete the test file afterwards.
3. **Free space.** `df -h /Volumes/<name>`. Require the planned total plus 10 percent.
4. **Name test.** On NTFS or exFAT, try creating a file whose name contains `:` or a trailing dot in a scratch folder. If it fails, the name-safety rules in `offload-protocol.md` are mandatory.

## Metadata files

On non-Mac file systems macOS writes `._name` files (AppleDouble) and sometimes `.DS_Store` while copying. They hold extended attributes, which this protocol does not promise to keep, so the ones macOS generated can go. But the drive can also hold real `._name` files: ones that were there before the run, and ones copied from the Mac as user data.

Being absent from the manifest does not prove a file was generated. The copy phase of the reference implementation therefore adds every `._name` file already in the archive root to a cumulative protected list (`Transfer-Logs/appledouble-before.json`) at the start of every copy run, including reruns and later batches. After the run, remove a `._name` file only if **all** of these hold:

- It is not in that protected list.
- It is not a destination in the manifest (so it is not copied user data).
- Its sibling `name` is a file this run wrote (`verified` or `archive-verified` in the manifest).
- It is a regular file, not a symlink, found by walking the archive root without following links.
- It starts with the AppleDouble magic number `00 05 16 07`.

Anything else is left alone, including generated files from an earlier batch that was not cleaned up before the next one started. Leftover AppleDouble files are harmless.

```python
import json, os, stat
from pathlib import Path

root = Path("/Volumes/<name>/<archive-root>")   # the same root path the offload script used
logs = root / "Transfer-Logs"
before_file = logs / "appledouble-before.json"
if not before_file.exists():
    raise SystemExit("no pre-run list of AppleDouble files; leave them, they are harmless")
with open(before_file, encoding="utf-8") as f:
    before = {os.path.normpath(p) for p in json.load(f)}
with open(logs / "manifest.jsonl", encoding="utf-8") as f:
    recs = [json.loads(l) for l in f if l.strip()]
dests = {os.path.normpath(r["dest"]) for r in recs if r.get("dest")}
written = {os.path.normpath(r["dest"]) for r in recs if r["status"] in ("verified", "archive-verified")}
removed = 0
for dirpath, _dirs, files in os.walk(root):     # does not follow symlinked folders
    for name in files:
        if not name.startswith("._"):
            continue
        p = os.path.normpath(os.path.join(dirpath, name))
        if p in before or p in dests or os.path.join(os.path.dirname(p), name[2:]) not in written:
            continue
        if not stat.S_ISREG(os.lstat(p).st_mode):
            continue
        fd = os.open(p, os.O_RDONLY | os.O_NOFOLLOW)
        try:
            magic = os.read(fd, 4)
        finally:
            os.close(fd)
        if magic == b"\x00\x05\x16\x07":
            os.unlink(p); removed += 1
print("removed", removed, "AppleDouble files")
```

Run it only after the delete phase, when no copy is in progress.

## Disconnect safely

1. Make sure no copy job is still running and nothing is open on the drive: `lsof +D /Volumes/<name> 2>/dev/null`. Finder windows can hold the volume; closing them is enough.
2. Flush: `sync`.
3. Unmount through whatever mounted it:
   - `anylinuxfs unmount /Volumes/<name> --wait-for-vm` for the NTFS helper. Always name the mount point: without it, `anylinuxfs unmount` unmounts every drive it manages. Then confirm with `mount | grep /Volumes/<name>`.
   - `diskutil unmount /Volumes/<name>` for native mounts.
4. Eject the whole disk: `diskutil eject diskN`.
5. After ejecting, `diskutil list` may still show the disk while the cable is attached. That is normal as long as no partition is mounted (`diskutil info diskNsM | grep Mounted` reports `No`). Then unplug.

Tell the user how to mount the drive again next time, including the exact command if a helper is needed.
