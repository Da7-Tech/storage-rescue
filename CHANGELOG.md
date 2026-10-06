# Changelog

## 1.0.0 (2026-10-07)

First public release.

- Two modes: Offload (verified move to an external drive, then deletion from the Mac) and Reclaim (deletion of caches and of what the user decides to remove).
- Every move and deletion goes through a question the user answers, asked with the agent's question tool.
- Offload verifies each file by SHA-256 right after copying and again after remounting the drive, re-hashes the source before deleting it, and deletes only verified entries.
- References for NTFS, exFAT, FAT32, and APFS drives, a catalog of safe cleanup commands, app data, and "System Data".
