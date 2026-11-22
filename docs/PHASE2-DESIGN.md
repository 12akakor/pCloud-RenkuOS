# Phase 2 design: writing without losing data

This document is a design, not shipped behaviour. The current image is still
strictly read-only: every mutating FUSE operation returns `EROFS`. The
components described here are being added and unit-tested before the add-on
is allowed to change anything remotely.

## Invariant

**No remote mutation may be attempted until the intent is durable locally,
and no local state may be discarded until the remote mutation is confirmed.**

## Write-ahead journal

`PCloudJournal` is an append-only file of records: a 16-byte header
(magic `PCJ1`, type, length, CRC32C) followed by a flattened `BMessage`.

- A record is appended with one `write()` followed by `fsync()`.
- Replay verifies magic, length and CRC32C and stops at the first bad or
  truncated record. A crash can only leave a torn tail; earlier records stay
  valid.
- `Clear()` truncates the journal only after all recorded operations are
  resolved.

## Operation ordering

### Creating a folder
1. Journal `{op: mkdir, parent, name, tempName}` — fsync.
2. pCloud `createfolder` with a temporary name such as `.pcloudfs-tmp-<uuid>`.
3. Verify the returned metadata (parent, name).
4. pCloud `renamefolder` to the final name.
5. Journal `{op: mkdir-done, fileid}` — fsync.
6. Update the local cache atomically.

A failure at step 2 or 4 leaves a temporary remote folder but never loses
user data; replay retries.

### Uploading a file
1. Write the complete local file to a temp file, fsync it, compute SHA-1.
2. Journal `{op: upload, path, size, sha1, tempName}` — fsync.
3. pCloud `uploadfile` under a temporary name.
4. `checksumfile` on the uploaded temporary file; compare SHA-1 and size.
5. pCloud `renamefile` to the final name.
6. Journal `{op: upload-done, fileid}` — fsync.
7. Promote the cache entry atomically.

A failed step 3/4/5 never destroys the original: the local temp file and the
journal entry remain until the whole sequence succeeds.

### Modifying an existing file
Never overwrite in place remotely. Upload to a temporary name, verify, then
rename over the target only after the new object is fully confirmed.

### Deleting
1. Journal `{op: delete, fileid, path}` — fsync.
2. pCloud `deletefile` / `deletefolder` (moves to pCloud trash, recoverable).
3. Journal `{op: delete-done, fileid}` — fsync.
4. Only then remove the local cache copy.

### Renaming
Journal the old and new ids, rename remotely, verify with `stat`, then update
the cache. Replay detects whether the old or new id exists and completes or
reverses the operation.

## Conflicts and offline behaviour

- While offline, writes stay in the journal as pending operations; the local
  cache keeps the authoritative dirty copy.
- On reconnect, operations replay in order. A conflict (remote object changed
  since it was cached) is surfaced to the user; nothing is silently
  overwritten.
- The read-only phase has none of these states, so there is no chance of
  corrupt remote data today.

## Before enabling writes

- [x] Journal with CRC32C and replay
- [x] Journal unit tests (append, replay, torn tail, clear)
- [ ] createfolder, uploadfile, renamefile/folder, deletefile/folder wrappers
- [ ] FUSE write handlers that journal before replying
- [ ] Replay test against a mock account
- [ ] Only then: enable writes in a new test image
