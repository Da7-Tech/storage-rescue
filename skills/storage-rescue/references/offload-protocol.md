# Offload protocol

The goal: every byte that leaves the Mac exists on the drive, verified, before the Mac copy is deleted. The manifest is the proof.

The run has three phases, and sources are deleted only in the last one:

1. **Copy phase.** Copy or pack each job, verify each file by reading it back with the Mac's cache bypassed, and record the job's source inventory. Nothing is deleted.
2. **Re-verification phase.** Unmount the drive, mount it again, then re-hash every destination in the manifest. Remounting clears caches that `F_NOCACHE` cannot bypass, such as the page cache of a helper VM that serves an NTFS drive over NFS.
3. **Delete phase.** For each job whose files all passed re-verification, re-read the source inventory and re-hash every source file against the hash its copy was verified with. If anything differs, keep the whole source and report it. Otherwise delete only the entries recorded in the inventory, re-checking each one just before removal, and remove folders only when they are empty. Anything that appeared later stays.

Copying does not use space on the Mac, so deferring deletion to the end costs nothing except time.

## 1. Preconditions

- The drive is mounted read-write and tested (`external-drives.md`).
- The destination root is on the external drive, not the Mac: `stat -f %d "$ROOT"` differs from `stat -f %d "$HOME"`.
- Free space on the drive is at least the planned total plus 10 percent.
- The user approved the categories and the item list.
- No planned source is open by a running process: `lsof +D <folder>` for folders, `lsof <file>` for files. Skip anything in use and tell the user. Ask the user not to work in those folders until the run ends.
- Nothing modified in the last 24 hours is moved unless the user named it. Recent files are usually active work.

## 2. Metadata that is not preserved

The reference implementation preserves file contents and modification times. Archives also preserve symlinks, permissions, and empty folders. **Neither mode preserves extended attributes, ACLs, resource forks, or Finder tags.** NTFS and exFAT cannot store most of them anyway.

Before the copy phase, check every source completely. Do not sample, and do not hide errors:

```bash
W="$(mktemp -d "${TMPDIR:-/tmp}/sr-meta-XXXXXX")" || { echo "cannot create a work folder"; exit 1; }
xattr -r "$SRC" > "$W/xattrs.txt"; echo "xattr exit: $?"
awk -F': ' '{print $NF}' "$W/xattrs.txt" | sort | uniq -c | sort -rn     # attribute names and counts
find "$SRC" -acl -print > "$W/acls.txt"; echo "find exit: $?"
wc -l < "$W/acls.txt"                                                     # items with ACLs
```

The results go into a new private folder, so the check never writes over an existing file.

A nonzero exit means some items could not be read, so the lists are incomplete. Resolve that (permissions, Full Disk Access) or drop the unreadable items from the plan before going on.

Usually harmless to lose: `com.apple.quarantine`, `com.apple.provenance`, `com.apple.macl`, `com.apple.lastuseddate#PS`, `com.apple.metadata:kMDItemWhereFroms`. Worth asking about: `com.apple.metadata:_kMDItemUserTags` (Finder tags), `com.apple.FinderInfo`, `com.apple.ResourceFork` (old Mac files, fonts, aliases), app-specific attributes, and ACLs (inspect with `ls -led <path>`).

Show the user the attribute names and counts and the ACL count. Any item whose metadata goes beyond the harmless list needs the user's explicit OK to lose that metadata. Without it, remove the item from the plan and keep it on the Mac, or use an APFS or HFS+ drive with a tool that preserves metadata (`ditto`) and an added check that compares `xattr -l` and `ls -le` output after copying.

## 3. Destination layout

Create one root folder on the drive (for example `Mac-Archive/`) with one folder per category: `Projects/`, `AI-Models/`, `Movies/`, `Series/`, `Recordings/`, `Images/`, `PDFs/`, `Books/`, `Installers/`, `Backups/<app>/`, `Transfer-Logs/`. Use the user's language for folder names if they prefer. Mirror enough of the original path that the user can tell where each item came from.

Write a plain-text `README.txt` at the root that explains, in the user's language: what each folder holds, where it came from on the Mac, how to restore an item, and that this is the only copy.

## 4. Plan file

Write a JSON plan before copying. Each job has `src`, `dest` (relative to the root), `mode`, and `category`. Show the plan totals to the user.

| Mode | Use for | Verification |
|---|---|---|
| `copy` | Media, models, installers, single large files, folders that contain only regular files | Per-file SHA-256 of the source compared with a cache-bypassing read of the destination |
| `tgz` | Project folders with thousands of small files, or anything with symlinks (`node_modules`, `.git`, build output) | Pack to `.tar.gz`, then stream the archive back from the drive and compare every member's type, size, and SHA-256 with what was packed |
| `tar` | Large, already-compressed folders that contain symlinks (model snapshots) | Same as `tgz` without compression |

`copy` refuses a folder that contains symlinks, FIFOs, sockets, or devices. Use `tgz` or `tar` for those. Archives refuse sockets and devices, which cannot be archived meaningfully. Also use an archive mode when names on the source cannot be stored on the destination file system.

## 5. Name safety for NTFS and exFAT

For `copy` destinations, make each path component safe:

- Normalize to Unicode NFC (macOS often stores decomposed forms).
- Replace `< > : " \ | ? *` and control characters with `_`.
- Strip trailing spaces and dots, then append `_` if anything was stripped.
- Prefix reserved names (`CON`, `PRN`, `AUX`, `NUL`, `COM1` to `COM9`, `LPT1` to `LPT9`) with `_`, with or without an extension.
- If the target exists with identical content (same size and SHA-256), record it as already present and verified. If it exists with different content, add ` (2)`, ` (3)`, and so on. Never overwrite an existing file.

The manifest records the source and the final destination, so renamed items stay traceable.

## 6. Copy and verify, per file

1. Refuse if the destination is a symlink, resolves outside the drive root, or is the same file as the source.
2. Note the source's size, modification time, inode, and change time (ctime).
3. Read the source in chunks, hashing as you go, and write to a new, randomly named temporary file in the destination folder. It is created exclusively; if the name is taken, pick another. Cleanup on failure deletes the temporary file only if it is still the file this step created, so an existing file is never opened, replaced, or deleted.
4. `fsync` the temporary file.
5. Stat the source again. If size, modification time, inode, or ctime changed, delete the temporary file and fail the job.
6. Hash the temporary file again from the drive with the cache disabled (`fcntl F_NOCACHE` on macOS, value 48).
7. If the hashes match, move it to the final name with a rename that cannot replace an existing file (`renamex_np` with `RENAME_EXCL`). If the name was taken in the meantime, use the next ` (n)` name. Then restore the modification time and append a manifest line. If the hashes differ, delete the temporary file and fail the job.

File systems without exclusive rename (exFAT) skip the temporary file: the final file is created exclusively under its own name and written directly, so nothing is ever replaced. If such a copy fails, the file is not deleted automatically, because another process could have written to it; it is recorded as `unverified-leftover`. If the run is interrupted, a partial file can also remain under a final name without any record. Neither has a `verified` record in the manifest; list such files for the user rather than trusting or deleting them.

## 7. Manifest

One JSON object per line (JSONL), appended and fsynced per record, written both to the drive (`Transfer-Logs/manifest.jsonl`) and to a local log folder (for example `~/Library/Logs/storage-rescue/`).

| Status | Meaning |
|---|---|
| `verified` | A file copied and verified (`job`, `src`, `dest`, `size`, `sha256`) |
| `identical-already-present-verified` | The destination already held identical bytes |
| `archive-verified` | An archive packed and verified (`members`, `source_bytes`, `files` map of member to SHA-256, `sha256` of the archive) |
| `job-copied` | Every file of the job verified. Holds the source `inventory` taken before copying |
| `reverified` / `reverify-failed` | Result of the re-verification phase for one destination |
| `source-deleted` / `source-kept` | Result of the delete phase for one job, with the reason when kept |
| `source-partly-deleted` | Verified entries were deleted, but entries that changed or appeared after verification were left in place (listed in `left`) |
| `unverified-leftover` | On a file system without exclusive rename, a directly written file whose copy failed. It was not deleted automatically; show it to the user |

Use one manifest per archive root. Later batches append to the same file; jobs that already have a `job-copied` record are skipped.

Re-open the manifest for every write. On helper-VM or network mounts, long-held descriptors go stale (`ESTALE`, errno 70); retry the write a few times with a short pause.

## 8. Run it detached

Long copies outlive an agent's command timeout and can be killed by the host. Start each phase in the background (`nohup python3 offload.py copy > run.log 2>&1 &`, or the host's background-process feature), then poll `run.log`. A rerun of the copy phase skips jobs that already have a `job-copied` record.

## 9. Reference implementation (Python 3.9 or later, standard library)

Adapt the paths and the job list. Keep the checks intact.

```python
import ctypes, errno, fcntl, gzip, hashlib, json, os, secrets, stat, tarfile, time, unicodedata
from pathlib import Path

CHUNK = 8 * 1024 * 1024
F_NOCACHE = 48
RENAME_EXCL = 0x4  # renamex_np flag: fail with EEXIST instead of replacing the destination
_libc = ctypes.CDLL(None, use_errno=True)
BAD = {c: "_" for c in '<>:"\\|?*'}
RESERVED = {"CON", "PRN", "AUX", "NUL", *(f"COM{i}" for i in range(1, 10)), *(f"LPT{i}" for i in range(1, 10))}
VERIFIED = ("verified", "identical-already-present-verified", "archive-verified")

# ---------- helpers

def safe_name(name):
    name = unicodedata.normalize("NFC", name)
    name = "".join(BAD.get(c, c) if ord(c) >= 32 else "_" for c in name)
    stripped = name.rstrip(" .")
    if stripped != name:
        name = stripped + "_"
    if name.split(".")[0].upper() in RESERVED:
        name = "_" + name
    return name or "_"

def hash_nocache(path):
    h = hashlib.sha256()
    fd = os.open(path, os.O_RDONLY | os.O_NOFOLLOW)
    try:
        fcntl.fcntl(fd, F_NOCACHE, 1)
        while True:
            b = os.read(fd, CHUNK)
            if not b:
                break
            h.update(b)
    finally:
        os.close(fd)
    return h.hexdigest()

def hash_file(path):
    h = hashlib.sha256()
    with open(path, "rb") as f:
        while True:
            b = f.read(CHUNK)
            if not b:
                break
            h.update(b)
    return h.hexdigest()

def record(manifests, obj):
    line = json.dumps(obj, ensure_ascii=False) + "\n"
    for m in manifests:
        for attempt in range(6):
            try:
                with open(m, "a", encoding="utf-8") as f:
                    f.write(line); f.flush(); os.fsync(f.fileno())
                break
            except OSError as e:
                if e.errno not in (17, 70) or attempt == 5:
                    raise
                time.sleep(5)

def read_manifest(path):
    with open(path, encoding="utf-8") as f:
        return [json.loads(l) for l in f if l.strip()]

def unique(p):
    if not p.exists() and not p.is_symlink():
        return p
    suffix = ".tar.gz" if p.name.endswith(".tar.gz") else p.suffix
    stem = p.name[: len(p.name) - len(suffix)] if suffix else p.name
    i = 2
    while True:
        q = p.with_name(f"{stem} ({i}){suffix}")
        if not q.exists() and not q.is_symlink():
            return q
        i += 1

def check_dest(dst, root):
    real_root = os.path.realpath(root)
    parent = os.path.realpath(dst.parent)
    if os.path.commonpath([parent, real_root]) != real_root:
        raise IOError(f"destination resolves outside the drive root: {dst}")
    if dst.is_symlink():
        raise IOError(f"destination is a symlink: {dst}")

def create_temp(dst):
    """Create a new temporary file next to dst. Never opens or replaces an existing file."""
    for _ in range(100):
        part = dst.with_name(f"{dst.name}.sr-{secrets.token_hex(6)}.part")
        try:
            fd = os.open(part, os.O_WRONLY | os.O_CREAT | os.O_EXCL | os.O_NOFOLLOW, 0o666)
        except FileExistsError:
            continue
        st = os.fstat(fd)
        return part, os.fdopen(fd, "wb"), (st.st_dev, st.st_ino)
    raise IOError(f"could not create a temporary file next to {dst}")

def drop_temp(part, ident):
    """Delete the temporary file only if it is still the one create_temp made."""
    try:
        st = os.lstat(part)
    except FileNotFoundError:
        return
    if (st.st_dev, st.st_ino) == ident:
        os.unlink(part)

_excl_rename = {}

def has_excl_rename(folder):
    """Probe once per file system whether renamex_np(RENAME_EXCL) works there (exFAT: no)."""
    dev = os.stat(folder).st_dev
    if dev not in _excl_rename:
        a, fo, ident = create_temp(Path(folder) / ".sr-probe")
        fo.close()
        b = Path(folder) / f".sr-probe-{secrets.token_hex(6)}.part"
        ok = _libc.renamex_np(os.fsencode(a), os.fsencode(b), RENAME_EXCL) == 0
        err = ctypes.get_errno()
        if ok:
            os.unlink(b)  # created by our own exclusive rename
        else:
            drop_temp(a, ident)
            if err not in (errno.ENOTSUP, errno.EINVAL):
                raise OSError(err, os.strerror(err), str(folder))
        _excl_rename[dev] = ok
    return _excl_rename[dev]

def create_final(dst):
    """Create the final file itself exclusively, at dst or the next free ' (n)' name."""
    wanted = dst
    for _ in range(1000):
        dst = unique(wanted)
        try:
            fd = os.open(dst, os.O_WRONLY | os.O_CREAT | os.O_EXCL | os.O_NOFOLLOW, 0o666)
        except FileExistsError:
            continue
        st = os.fstat(fd)
        return dst, os.fdopen(fd, "wb"), (st.st_dev, st.st_ino)
    raise IOError(f"no free destination name for {wanted}")

def open_output(dst):
    """Returns (path, file, identity, direct). With exclusive rename, write a temporary file and
    rename it at the end. Without it, write the final file directly, so nothing is ever replaced."""
    if has_excl_rename(dst.parent):
        part, fo, ident = create_temp(dst)
        return part, fo, ident, False
    path, fo, ident = create_final(dst)
    return path, fo, ident, True

def publish(part, dst):
    """Rename part to dst, failing with FileExistsError instead of replacing an existing file."""
    if _libc.renamex_np(os.fsencode(part), os.fsencode(dst), RENAME_EXCL) != 0:
        err = ctypes.get_errno()
        raise OSError(err, os.strerror(err), str(dst))

def finish(part, dst):
    """Publish part under dst or, if that name is taken, the next free ' (n)' name. Returns the final path."""
    wanted = dst
    for _ in range(1000):
        dst = unique(wanted)
        try:
            publish(part, dst)
            return dst
        except FileExistsError:
            continue
    raise IOError(f"no free destination name for {wanted}")

def give_up(job, path, ident, direct, manifests):
    """Clean up after a failed copy. A random temporary name is deleted if it is still ours.
    A final name written directly is never deleted, because another process could have written to it;
    it is recorded so the user can review it."""
    if not direct:
        drop_temp(path, ident)
        return
    print("UNVERIFIED FILE LEFT ON THE DRIVE", path, flush=True)
    try:
        record(manifests, {"job": job, "dest": str(path), "status": "unverified-leftover"})
    except OSError:
        pass

def _raise(err):
    raise err

def _entry(p, st):
    m = st.st_mode
    kind = "f" if stat.S_ISREG(m) else "d" if stat.S_ISDIR(m) else "l" if stat.S_ISLNK(m) else "o"
    if kind == "d":
        return [kind, 0, 0, st.st_ino, 0, ""]  # a folder's own times change whenever its entries do
    # ctime cannot be set back by programs, so it catches edits that restore size and mtime
    return [kind, st.st_size, st.st_mtime_ns, st.st_ino, st.st_ctime_ns, os.readlink(p) if kind == "l" else ""]

def inventory(src):
    """Every entry under src with type, size, mtime, inode, ctime and link target. Raises on any unreadable folder."""
    src = Path(src)
    top = os.lstat(src)
    if not stat.S_ISDIR(top.st_mode):
        return {".": _entry(src, top)}
    inv = {}
    for root, dirs, files in os.walk(src, onerror=_raise):  # does not descend into symlinked dirs
        dirs.sort()
        for name in dirs + sorted(files):
            p = os.path.join(root, name)
            inv[os.path.relpath(p, src)] = _entry(p, os.lstat(p))
    return inv

# ---------- copy mode

def copy_verified(job, src, dst, root, manifests):
    dst.parent.mkdir(parents=True, exist_ok=True)
    check_dest(dst, root)
    st0 = os.lstat(src)
    if not stat.S_ISREG(st0.st_mode):
        raise IOError(f"not a regular file: {src}")
    if dst.exists():
        if os.path.samefile(src, dst):
            raise IOError(f"destination is the source itself: {dst}")
        if dst.stat().st_size == st0.st_size:
            src_sha = hash_file(src)
            if hash_nocache(dst) == src_sha:
                record(manifests, {"job": job, "src": str(src), "dest": str(dst), "size": st0.st_size,
                                   "sha256": src_sha, "status": "identical-already-present-verified"})
                return
        dst = unique(dst)
    part, fo, ident, direct = open_output(dst)
    h = hashlib.sha256()
    try:
        with open(src, "rb") as fi, fo:
            while True:
                b = fi.read(CHUNK)
                if not b:
                    break
                h.update(b); fo.write(b)
            fo.flush(); os.fsync(fo.fileno())
            st = os.fstat(fo.fileno())
            ident = (st.st_dev, st.st_ino)  # FAT32 assigns the inode number once data is written
        st1 = os.lstat(src)
        if _entry(src, st1) != _entry(src, st0):
            raise IOError(f"source changed during copy: {src}")
        if part.stat().st_size != st0.st_size or hash_nocache(part) != h.hexdigest():
            raise IOError(f"verification failed: {src}")
        dst = part if direct else finish(part, dst)
    except BaseException:
        give_up(job, part, ident, direct, manifests)
        raise
    os.utime(dst, ns=(st0.st_atime_ns, st0.st_mtime_ns))
    record(manifests, {"job": job, "src": str(src), "dest": str(dst), "size": st0.st_size,
                       "sha256": h.hexdigest(), "status": "verified"})

def copy_job(job, src, dst_root, root, inv, manifests):
    special = [k for k, v in inv.items() if v[0] not in ("f", "d")]
    if special:
        raise IOError(f"symlinks or special files, use tgz or tar mode: {special[:5]}")
    if inv.get(".", [""])[0] == "f":
        copy_verified(job, src, dst_root, root, manifests)
        return
    for rel, (kind, *_rest) in inv.items():
        out = dst_root.joinpath(*[safe_name(p) for p in Path(rel).parts])
        if kind == "d":
            out.mkdir(parents=True, exist_ok=True)
        else:
            copy_verified(job, src / rel, out, root, manifests)

# ---------- archive mode

class _Hashing:
    def __init__(self, f): self.f, self.h = f, hashlib.sha256()
    def read(self, n=-1):
        b = self.f.read(n); self.h.update(b); return b

class _NoCache:
    def __init__(self, path):
        self.fd = os.open(path, os.O_RDONLY | os.O_NOFOLLOW); fcntl.fcntl(self.fd, F_NOCACHE, 1)
    def read(self, n=-1): return os.read(self.fd, CHUNK if n is None or n < 0 else n)
    def close(self): os.close(self.fd)

def _kind(t):
    return "f" if t.isreg() else "d" if t.isdir() else "l" if t.issym() else "h" if t.islnk() else "p" if t.isfifo() else "o"

def pack_job(job, src, dest, root, inv, manifests, compress=True):
    dest.parent.mkdir(parents=True, exist_ok=True)
    dest = unique(dest)
    check_dest(dest, root)
    part, out, ident, direct = open_output(dest)
    expected, total = {}, 0
    entries = [src] + [src / rel for rel in inv if rel != "."]
    try:
        with out:
            with tarfile.open(fileobj=out, mode="w|gz" if compress else "w|", format=tarfile.PAX_FORMAT) as tar:
                for p in entries:
                    arc = src.name if p == src else str(Path(src.name) / p.relative_to(src))
                    ti = tar.gettarinfo(str(p), arcname=arc)
                    if ti is None:
                        raise IOError(f"cannot archive socket or device: {p}")
                    ti.uname = ti.gname = ""
                    if ti.isreg():
                        with open(p, "rb") as f:
                            hr = _Hashing(f); tar.addfile(ti, hr)
                        expected[arc] = ("f", ti.size, hr.h.hexdigest()); total += ti.size
                    else:
                        tar.addfile(ti); expected[arc] = (_kind(ti), 0, ti.linkname)
            out.flush(); os.fsync(out.fileno())
            st = os.fstat(out.fileno())
            ident = (st.st_dev, st.st_ino)  # FAT32 assigns the inode number once data is written
        seen = {}
        reader = _NoCache(part)
        try:
            raw = gzip.GzipFile(fileobj=reader) if compress else reader
            with tarfile.open(fileobj=raw, mode="r|") as tar:
                for m in tar:
                    if m.isreg():
                        f, h = tar.extractfile(m), hashlib.sha256()
                        while True:
                            b = f.read(CHUNK)
                            if not b:
                                break
                            h.update(b)
                        seen[m.name] = ("f", m.size, h.hexdigest())
                    else:
                        seen[m.name] = (_kind(m), 0, m.linkname)
        finally:
            reader.close()
        if seen != expected:
            raise IOError(f"archive verification failed: {src}")
        dest = part if direct else finish(part, dest)
    except BaseException:
        give_up(job, part, ident, direct, manifests)
        raise
    files = {k: v[2] for k, v in expected.items() if v[0] == "f"}
    for k, (kind, _size, target) in expected.items():
        if kind == "h":  # a hard link member carries no data; its content is the verified target's
            files[k] = files[target]
    record(manifests, {"job": job, "src": str(src), "dest": str(dest), "size": dest.stat().st_size,
                       "sha256": hash_nocache(dest), "status": "archive-verified",
                       "members": len(expected), "source_bytes": total, "files": files})

# ---------- phase 1: copy (deletes nothing)

def note_appledouble(root):
    """At the start of every copy run, add the ._ files already in the archive root to a cumulative
    protected list, so metadata cleanup never touches a file that existed before any run."""
    out = root / "Transfer-Logs" / "appledouble-before.json"
    out.parent.mkdir(parents=True, exist_ok=True)
    known = set()
    if out.exists():
        with open(out, encoding="utf-8") as f:
            known = set(json.load(f))
    known |= {os.path.join(d, n) for d, _s, fs in os.walk(root, onerror=_raise) for n in fs if n.startswith("._")}
    tmp = out.with_name(f"{out.name}.{secrets.token_hex(6)}.tmp")
    with open(tmp, "x", encoding="utf-8") as f:
        json.dump(sorted(known), f, ensure_ascii=False); f.flush(); os.fsync(f.fileno())
    os.replace(tmp, out)

def run_copy_phase(jobs, root, manifests):
    note_appledouble(root)
    done = {r["job"] for r in read_manifest(manifests[0]) if r["status"] == "job-copied"} if manifests[0].exists() else set()
    for j in jobs:
        job, src = j["src"], Path(j["src"])
        if job in done:
            continue
        try:
            inv = inventory(src)
            dest = root.joinpath(*[safe_name(p) for p in Path(j["dest"]).parts])
            if j["mode"] == "copy":
                copy_job(job, src, dest, root, inv, manifests)
            else:
                pack_job(job, src, dest, root, inv, manifests, compress=j["mode"] == "tgz")
            if inventory(src) != inv:
                raise IOError("source changed during the job")
            record(manifests, {"job": job, "status": "job-copied", "inventory": inv})
        except Exception as e:
            record(manifests, {"job": job, "status": "failed", "error": str(e)})
            print("FAILED", job, e, flush=True)

# ---------- phase 2: re-verify after unmounting and remounting the drive

def run_reverify_phase(manifests):
    for r in read_manifest(manifests[0]):
        if r["status"] not in VERIFIED:
            continue
        try:
            ok = hash_nocache(r["dest"]) == r["sha256"]
        except OSError:
            ok = False
        record(manifests, {"job": r["job"], "dest": r["dest"], "status": "reverified" if ok else "reverify-failed"})

# ---------- phase 3: delete sources whose job fully passed

def source_hashes(recs):
    """Source file path -> the SHA-256 its drive copy was verified against."""
    out = {}
    for r in recs:
        if r["status"] in ("verified", "identical-already-present-verified"):
            out[r["src"]] = r["sha256"]
        elif r["status"] == "archive-verified":
            parent = Path(r["job"]).parent
            for arc, sha in r["files"].items():
                out[str(parent / arc)] = sha
    return out

def why_keep(src, inv, hashes):
    if inventory(src) != inv:
        return "source changed since it was copied"
    for rel, e in inv.items():
        if e[0] == "f":
            p = src if rel == "." else src / rel
            if hashes.get(str(p)) != hash_file(p):
                return f"content differs from the verified copy: {p}"
    return None

def _remove(p, is_dir, inside_job):
    rm = os.rmdir if is_dir else os.unlink
    try:
        rm(p)
    except PermissionError:  # immutable flag, or a read-only folder inside the job
        os.chflags(p, 0, follow_symlinks=False)
        if inside_job:
            os.chmod(os.path.dirname(p), 0o700)
        rm(p)

def _still_verified(p, recorded, hashes):
    now = _entry(p, os.lstat(p))
    if now == list(recorded):
        return True
    # Deleting one name of a hard-linked file changes the other name's ctime. Accept a
    # ctime-only difference only if the content still has the verified hash.
    same_but_ctime = now[:4] + now[5:] == list(recorded[:4]) + list(recorded[5:])
    return same_but_ctime and now[0] == "f" and hash_file(p) == hashes.get(str(p))

def remove_recorded(src, inv, hashes):
    """Delete only entries recorded in inv, each re-checked just before removal.
    Folders are removed only when empty, so anything new survives. Returns what was left."""
    if "." in inv:
        items = [(".", src)]
    else:
        deepest_first = sorted(inv, key=lambda r: len(Path(r).parts), reverse=True)
        items = [(rel, src / rel) for rel in deepest_first] + [(None, src)]
    left = []
    for rel, p in items:
        try:
            if rel is None or inv[rel][0] == "d":
                _remove(p, True, rel is not None)
            elif not _still_verified(p, inv[rel], hashes):
                left.append(f"{rel}: changed")
            else:
                _remove(p, False, rel != ".")
        except OSError as e:
            left.append(f"{rel or '.'}: {e.strerror}")
    return left

def run_delete_phase(manifests):
    recs = read_manifest(manifests[0])
    hashes = source_hashes(recs)
    dests = {}
    for r in recs:
        if r["status"] in VERIFIED:
            dests.setdefault(r["job"], set()).add(r["dest"])
    passed = {r["dest"] for r in recs if r["status"] == "reverified"}
    failed = {r["dest"] for r in recs if r["status"] == "reverify-failed"}
    for r in recs:
        if r["status"] != "job-copied":
            continue
        job, src = r["job"], Path(r["job"])
        if not src.exists() and not src.is_symlink():
            continue
        mine = dests.get(job, set())
        if not mine or not mine <= passed or mine & failed:
            reason = "not every destination passed re-verification"
        else:
            reason = why_keep(src, r["inventory"], hashes)
        if reason is None:
            left = remove_recorded(src, r["inventory"], hashes)
            record(manifests, {"job": job, "status": "source-partly-deleted" if left else "source-deleted",
                               "left": left})
            if left:
                print("LEFT", job, left[:5], flush=True)
            continue
        record(manifests, {"job": job, "status": "source-kept", "reason": reason})
        print("KEPT", job, reason, flush=True)
```

Notes:

- `inventory()` compares lists, so it works the same on a freshly read JSON manifest.
- `manifests[0]` must be the drive copy, `root / "Transfer-Logs" / "manifest.jsonl"`, because the metadata cleanup in `external-drives.md` reads it.
- The delete phase reads every source file once more to re-hash it. On an internal SSD this is fast compared with the copy.
- An empty folder job has no file destinations, so the delete phase keeps it. Delete such folders by hand after checking they are empty.
- A `source-partly-deleted` job leaves only entries that were not verified. Show them to the user; they are still on the Mac.
- `pigz` can replace the built-in gzip for speed when packing; keep the read-back verification.
- Run each phase as a separate command so the drive can be unmounted and mounted again between phase 1 and phase 2.

## 10. After the run

1. **No sources left.** Every `job-copied` job has a `source-deleted`, `source-partly-deleted`, or `source-kept` record. Report kept and partly deleted jobs to the user, and any `unverified-leftover` files on the drive.
2. **Metadata cleanup.** On NTFS, exFAT, and FAT32 drives, macOS may write `._name` AppleDouble files while copying. See `external-drives.md` for the safe procedure, which deletes only files proven to be generated during the run.
3. **README and logs.** Update the drive README with this batch, and copy the run logs and plan into `Transfer-Logs/`.
4. **Measure.** `df -h /System/Volumes/Data` before and after.
5. **Warn.** The drive holds the only copy. Recommend a second backup (another drive or cloud) for anything irreplaceable.

## 11. Special cases

- **AI models an app is configured to use.** Read the app's config first. If a model is the app's active or primary model, keep it on the Mac or ask explicitly; moving it breaks the feature.
- **Chat and session databases of editors and agents.** Use the full-backup procedure in `app-data.md`; do not move a live database file.
- **Hugging Face caches.** The `hub/` layout uses symlinks into `blobs/`; use `tar` mode per model folder.
- **Folders of a few huge files inside a project.** Split: `copy` the huge files, `tgz` the rest, so verification stays fast and restoring a single file stays easy.
- **Duplicates of the same installer or video.** Hash them; move one copy and list the others as duplicates for the user to approve deleting.
